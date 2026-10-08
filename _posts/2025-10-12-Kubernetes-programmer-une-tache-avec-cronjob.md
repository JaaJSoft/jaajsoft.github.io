---
layout: article
title: "Kubernetes : Programmer des tâches avec CronJob"
description: "Programmer des tâches récurrentes dans Kubernetes avec un CronJob : chevauchements, historique, gestion des échecs, débogage et migration depuis cron."
tags:
  - kubernetes
  - cron
  - k8s
  - devops
author: Pierre Chopinet
---

Pour exécuter une tâche récurrente dans un cluster Kubernetes (une sauvegarde, un rapport, une purge), on utilise un CronJob : l'équivalent d'une ligne de crontab, qui crée un Job à chaque échéance. Voyons comment en écrire un, régler les options qui comptent vraiment (chevauchements, rattrapage, fuseau horaire, historique) et le déboguer quand il ne se lance pas.
<!--more-->

Dans cet article :
- Un CronJob minimal
- Ce qui change par rapport à cron
- Un CronJob plus complet
- Les options importantes
- Quelques variantes
- Lancer, suspendre et supprimer un CronJob
- Déboguer un CronJob
- Migrer une tâche cron vers Kubernetes

Pré-requis : un cluster Kubernetes (1.27 ou plus pour `spec.timeZone`) et `kubectl` configuré. Les deux manifestes complets de l'article ont été validés contre le schéma de l'API Kubernetes (versions 1.27 et 1.36). Si vous débutez avec cron côté Linux, lisez d'abord [Linux : Programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %}).

## Un CronJob minimal

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/5 * * * *"   # toutes les 5 minutes
  concurrencyPolicy: Forbid  # évite les chevauchements
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2         # réessais si échec (niveau Job)
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: hello
              image: busybox:1.36
              args: ["sh", "-c", "date; echo Hello from K8s CronJob"]
```

On l'applique, puis on suit ce qu'il crée : toutes les 5 minutes, le contrôleur crée un Job, qui crée lui-même un Pod.

```bash
kubectl apply -f cronjob.yaml
kubectl get cronjobs
kubectl get jobs
kubectl get pods
kubectl logs <pod>
```

## Ce qui change par rapport à cron

Le champ `schedule` utilise la syntaxe de cron (minute, heure, jour du mois, mois, jour de la semaine), avec les raccourcis `@hourly`, `@daily`, `@weekly`, `@monthly` et `@yearly`. Comme sous Linux, quand le jour du mois et le jour de la semaine sont tous les deux renseignés, il suffit que l'un des deux corresponde. Deux différences peuvent surprendre lors d'une migration : le jour de la semaine va de 0 à 6, et `7` pour le dimanche est refusé par l'API ; une variable `CRON_TZ` ou `TZ` dans `schedule` est aussi refusée, le fuseau horaire se règle avec le champ `timeZone`.

Pour le reste, la tâche ne tourne pas sur l'hôte mais dans un Pod, à partir d'une image. Les chevauchements se gèrent avec `concurrencyPolicy`, les exécutions passées restent visibles sous forme de Jobs (`successfulJobsHistoryLimit`, `failedJobsHistoryLimit`), et `startingDeadlineSeconds` décide si une exécution manquée, pendant une panne du contrôleur par exemple, doit encore être lancée. Sans `timeZone`, l'horaire est interprété dans le fuseau horaire du kube-controller-manager.

## Un CronJob plus complet

Voici un CronJob plus proche de ce qu'on met en production : un rapport lancé tous les jours à 2 h, heure de Paris, sans chevauchement, avec une durée maximale, des ressources limitées, un secret et un compte de service :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rapport-quotidien
spec:
  schedule: "0 2 * * *" # tous les jours à 02:00
  timeZone: Europe/Paris  # Kubernetes >= 1.27
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 120
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 2
  suspend: false
  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 900 # tue le job au-delà de 15 min
      template:
        spec:
          serviceAccountName: job-reporter
          restartPolicy: Never
          containers:
            - name: app
              image: ghcr.io/monorg/rapport:1.0.4
              imagePullPolicy: IfNotPresent
              env:
                - name: APP_ENV
                  value: production
                - name: DB_HOST
                  valueFrom:
                    secretKeyRef:
                      name: app-secrets
                      key: db_host
              resources:
                requests: { cpu: "100m", memory: "128Mi" }
                limits:   { cpu: "500m", memory: "512Mi" }
```

L'image porte un numéro de version précis plutôt que `latest`, pour savoir exactement ce qui tourne à chaque exécution. Le compte de service `job-reporter` n'est nécessaire que si la tâche a besoin de droits particuliers, pour appeler l'API Kubernetes par exemple (avec les règles RBAC correspondantes).

`restartPolicy` ne peut valoir que `OnFailure` ou `Never` dans un Job. Avec `OnFailure`, le conteneur est relancé dans le même Pod ; avec `Never`, chaque échec crée un nouveau Pod, ce qui garde les journaux des tentatives précédentes. La documentation de Kubernetes conseille d'ailleurs `Never` tant qu'on met la tâche au point.

Enfin, le nom d'un CronJob ne doit pas dépasser 52 caractères, car le contrôleur y ajoute 11 caractères pour nommer les Jobs.

## Les options importantes

- `schedule` : l'horaire, au format cron.
- `timeZone` : fuseau horaire IANA (`Europe/Paris`), stable depuis Kubernetes 1.27. Sans lui, c'est le fuseau du kube-controller-manager qui s'applique.
- `concurrencyPolicy` : `Allow` (par défaut) laisse les exécutions se chevaucher, `Forbid` saute la nouvelle exécution si la précédente tourne encore, `Replace` arrête la précédente pour lancer la nouvelle.
- `startingDeadlineSeconds` : délai maximal, en secondes, pour lancer une exécution qui a manqué son horaire. Au-delà, elle est sautée et comptée comme un échec. Pas de délai par défaut.
- `suspend` : à `true`, met la planification en pause, sans arrêter les Jobs déjà lancés.
- `successfulJobsHistoryLimit` et `failedJobsHistoryLimit` : nombre de Jobs terminés conservés, 3 et 1 par défaut.
- `jobTemplate.spec.backoffLimit` : nombre de nouvelles tentatives avant de considérer le Job en échec, 6 par défaut.
- `jobTemplate.spec.activeDeadlineSeconds` : durée maximale du Job. Une fois dépassée, ses Pods sont arrêtés et le Job échoue, même s'il restait des tentatives.
- `jobTemplate.spec.ttlSecondsAfterFinished` : supprime automatiquement le Job, et ses Pods, ce nombre de secondes après sa fin.

`Forbid` est le bon choix pour une tâche qui ne doit pas tourner deux fois en même temps, `Replace` pour une tâche courte qu'on préfère relancer. Mais aucune de ces options ne garantit une exécution unique : la documentation prévient que, dans certains cas, Kubernetes peut créer deux Jobs pour le même horaire, ou aucun. Les tâches doivent donc pouvoir être relancées sans dégâts (idempotentes).

Attention aussi à `startingDeadlineSeconds`. Avec une valeur inférieure à 10 secondes, le CronJob peut ne jamais se lancer, car le contrôleur ne vérifie les horaires que toutes les 10 secondes. À l'inverse, sans cette option, un contrôleur qui a manqué plus de 100 horaires (un CronJob qui tourne toutes les minutes sur un cluster arrêté pendant deux heures, par exemple) ne lance pas d'exécution de rattrapage et journalise l'erreur `too many missed start times` ; les exécutions suivantes ont lieu normalement. Sur un cluster qui ne tourne pas en continu, mieux vaut donc fixer un délai de rattrapage raisonnable.

## Quelques variantes

Toutes les 5 minutes, sans chevauchement :

```yaml
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
```

Le lundi à 9 h, en remplaçant le Job précédent s'il tourne encore, et en acceptant jusqu'à 5 minutes de retard :

```yaml
spec:
  schedule: "0 9 * * 1"
  concurrencyPolicy: Replace
  startingDeadlineSeconds: 300
```

Pour une tâche Python, le code et ses dépendances sont installés dans l'image : pas de venv à activer comme sous cron, on lance directement le module.

```yaml
containers:
  - name: job
    image: ghcr.io/monorg/worker:2.1.0
    args: ["python", "-m", "app.jobs.recalcule"]
```

## Lancer, suspendre et supprimer un CronJob

Pour lancer une exécution tout de suite, sans attendre l'horaire, on crée un Job à partir du modèle du CronJob :

```bash
kubectl create job rapport-manuel --from=cronjob/rapport-quotidien
```

Pour mettre la planification en pause, on passe `spec.suspend` à `true` dans le manifeste et on le réapplique (ou on le modifie avec `kubectl edit cronjob rapport-quotidien`), puis on remet `false` pour reprendre. Les horaires prévus pendant la pause comptent comme manqués : si le CronJob n'a pas de `startingDeadlineSeconds`, il lance une exécution dès qu'on le réactive, pour rattraper l'horaire manqué le plus récent.

Supprimer le CronJob supprime aussi les Jobs et les Pods qu'il a créés :

```bash
kubectl delete cronjob rapport-quotidien
```

## Déboguer un CronJob

```bash
kubectl describe cronjob rapport-quotidien
kubectl get jobs
kubectl describe job <job>
kubectl get pods --selector=job-name=<job>
kubectl logs <pod>    # ajouter -c <conteneur> s'il y en a plusieurs
kubectl get events -A | grep -i cronjob
```

`kubectl describe cronjob` affiche notamment la politique de concurrence, le `Starting Deadline Seconds`, la dernière exécution planifiée (`Last Schedule Time`, issue du champ `.status.lastScheduleTime`) et les événements récents.

Les Jobs créés par un CronJob s'appellent `<nom-du-cronjob>-<nombre>`, où le nombre est l'heure planifiée exprimée en minutes depuis 1970. Ils sont rattachés au CronJob par leurs `ownerReferences`, mais Kubernetes ne leur ajoute aucun label qui le désigne : pour pouvoir les filtrer, ajoutez vos propres labels dans `jobTemplate.metadata.labels`. Les Pods, eux, portent le label `job-name`, d'où le `--selector` ci-dessus.

Pour avoir une vue d'ensemble des horaires, la sortie JSON de `kubectl` se traite bien avec jq :

```bash
kubectl get cronjob -o json | jq '.items[] | {name: .metadata.name, schedule: .spec.schedule}'
```

Quelques problèmes reviennent souvent. Une image introuvable, un Secret ou un ConfigMap manquant se voient dans les événements du Pod (`Failed to pull image`, `not found`). Un Job qui ne se termine jamais se limite avec `activeDeadlineSeconds`, combiné à `concurrencyPolicy: Forbid` pour ne pas empiler les exécutions. Enfin, si les Jobs terminés s'accumulent, baissez les limites d'historique ou définissez `jobTemplate.spec.ttlSecondsAfterFinished` (le mécanisme de TTL est stable et actif par défaut depuis Kubernetes 1.23).

## Migrer une tâche cron vers Kubernetes

La tâche ne voit plus le système de fichiers de l'hôte : le script, ses dépendances et ses binaires doivent être dans l'image (ou montés en volume), il n'y a pas de `/usr/local/bin` de l'hôte. L'environnement ne s'hérite pas non plus : les variables se définissent dans `env`, depuis un ConfigMap ou un Secret, et les secrets ne doivent jamais apparaître dans les journaux.

Pour les droits, on donne au Pod un ServiceAccount avec le RBAC nécessaire, et on évite de le faire tourner en root avec un `securityContext`. Les fichiers à conserver d'une exécution à l'autre vont dans un volume persistant (`PersistentVolumeClaim`) : le système de fichiers du Pod disparaît avec lui, et un `emptyDir` ne vit pas plus longtemps que le Pod.

Côté journaux, écrire sur la sortie standard et la sortie d'erreur suffit le plus souvent : `kubectl logs` les affiche, et un collecteur comme Fluent Bit ou Grafana Alloy (qui remplace Promtail, en fin de vie depuis mars 2026) peut les envoyer vers Loki ou Elasticsearch si vous centralisez vos logs.

## Voir aussi

- [Linux : Programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %})
- [Kubernetes : Comment déployer un cluster k8s bare-metal avec k3s]({% post_url 2021-01-27-Comment-déployer-Kubernetes %})
- [Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %})
- [Documentation Kubernetes : CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Référence de l'API CronJob](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/cron-job-v1/)
- [Documentation Kubernetes : Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
