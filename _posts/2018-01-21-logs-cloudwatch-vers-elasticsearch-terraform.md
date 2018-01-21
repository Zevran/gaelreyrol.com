---
layout: post
title: "CloudWatch Logs vers ElasticSearch avec Terraform"
comments: true
description: "Comment rediriger ses logs CloudWatch vers ElasticSearch via une fonction Lambda d'AWS, le tout avec Terraform"
keywords: "logs, cloudwatch, elasticsearch, lambda, aws, terraform"
cover: /assets/images/2018-01-21/aws.png
---

Petit aparté avant de commencer ce petit tutoriel, si vous avez lu mon article [Reprise des hostilités](/2017/09/06/reprise-des-hostilites) vous remarquerez que je n'ai pas franchement tenu ma promesse ou qu'en tout cas je n'ai pas fait ce que je disais. En soi il n'y a pas mort d'homme mais j'aurai vraiment aimé prendre plus de temps pour vous partager mes expériences et mes connaissances.

Pour être un peu plus dans le vrai, je vais faire ce que je peux pour y arriver dès que j'ai du temps libre :)

---

## Comment rediriger ses logs CloudWatch vers ElasticSearch via une fonction Lambda d'AWS, le tout avec Terraform

### Introduction

Premier tutoriel en lien avec le DevOps, enfin !

Pour ce tutoriel je pars du principe que la maîtrise de **Terraform** et d'**Amazon Web Services** est déjà acquise.

Récemment je me suis lancé dans la centralisation des logs dit **journaux** en utilisant **Amazon Web Services**, pourquoi ?

Et bien pour ma part lorsque qu'un cluster disont **Kubernetes** pète totalement un câble et qu'il devient quasiment impossible d'utiliser son API pour lui faire reprendre ses esprits, la seule option qu'il vous reste pour arriver à vos fins est de jouer au détective dans les journaux qu'il a généré.

C'est là que le problème pourrai se compliquer si vous avez centralisé vos journaux à l'intérieur de votre cluster **Kubernetes**, centraliser les journaux c'est bien, tout centraliser au même endroit c'est moins bien.

J'ai bien pris soin de configurer les pilotes de journalisation **Docker** vers **CloudWatch** en utilisant **awslogs**. Vous pouvez voir [ici](https://docs.docker.com/engine/admin/logging/awslogs/) comment faire de même.

Une dernière chose va poser problème, fouiner dans ses journaux, on ne peux pas dire que **CloudWatch** soit vraiment fait pour ça, surtout quand des centaines/miliers sont générés par jour... Et nous connaissons tous une solution fantastique pour ça, **Elasticsearch** :)

### C'est parti !

Voilà, maintenant que le cadre est planté le tutoriel peut enfin démarrer !

Nous commençons donc avec deux ressources, **CloudWatch** et **ElasticSearch** :

```hcl
resource "aws_cloudwatch_log_group" "logs" {
  name              = "logs-du-cluster-k8s"
  retention_in_days = 14
}

resource "aws_elasticsearch_domain" "logs" {
  domain_name           = "logs-du-cluster-k8s"
  elasticsearch_version = "6.0"

  cluster_config {
    instance_type = "m4.large.elasticsearch"
  }

  ebs_options {
    ebs_enabled = true
    volume_type = "gp2"
    volume_size = 100
  }
}
```
Libre à vous de changer les paramètres de chaque ressource.

Il ne faut pas oublier la police d'accès au domaine **ElasticSearch** car sans celà nous ne pourrons pas intérragir avec son API.

```hcl
resource "aws_elasticsearch_domain_policy" "logs" {
  domain_name = "${aws_elasticsearch_domain.logs.domain_name}"

  access_policies = <<POLICIES
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "*"
        ]
      },
      "Action": [
        "es:*"
      ],
      "Resource": "${aws_elasticsearch_domain.logs.arn}/*"
    }
  ]
}
POLICIES
}
```

#### Lambda

Pour pouvoir rediriger les journaux **CloudWatch** vers **ElasticSearch** nous devons utiliser une fonction **Lambda** qui s'abonner aux émissions de journaux **CloudWatch** pour les retransmettre à **ElasticSearch** via son API.


