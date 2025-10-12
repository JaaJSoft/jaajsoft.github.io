---
layout: article
title: "Kubernetes : Programmer des tâches avec CronJob"
author: Pierre Chopinet
tags:
  - kubernetes
  - cronjob
  - k8s
  - jobs
  - devops
  - tutoriel
---

Besoin d’exécuter une commande ou un script de façon récurrente dans votre cluster Kubernetes, comme vous le feriez avec cron sur Linux ? Les CronJobs K8s sont faits pour ça.
Dans cet article, on voit comment créer un CronJob robuste, éviter les chevauchements, paramétrer l’historique, gérer les échecs et dépanner. On fera aussi le lien avec cron « classique » sur Linux pour migrer sereinement.

<!--more-->

> Si vous débutez avec cron côté Linux, lisez d’abord ce guide: [Linux : Programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %}). Pour installer rapidement un cluster K8s (k3s), voyez: [K8s : Comment déployer un cluster kubernetes bare-metal]({% post_url 2021-01-27-Comment-déployer-Kubernetes %}).

---

## TL;DR - le CronJob

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

- Appliquer : `kubectl apply -f cronjob.yaml`
- Lister : `kubectl get cronjobs` puis `kubectl get jobs` et `kubectl get pods`
- Logs : `kubectl logs <pod>`

---

## 1) CronJob vs cron Linux

Points communs :
- Syntaxe d’horaire type crontab: `* * * * *` (minute heure jour mois jour_sem)

Différences clés :
- Exécution dans un Pod (conteneurisé), pas sur l’hôte.
- Gestion des chevauchements via `concurrencyPolicy` (Allow/Forbid/Replace).
- Historique conservé via `successfulJobsHistoryLimit` et `failedJobsHistoryLimit`.
- `startingDeadlineSeconds` rattrape les exécutions manquées si le contrôleur était indisponible.
- Temps/fuseau: Kubernetes ≥ 1.27 supporte `spec.timeZone` (IANA, ex: `Europe/Paris`). Sinon, fuseau du contrôleur.

Pour réviser la syntaxe cron et les pièges, voyez l’article Linux mentionné plus haut.

---

## 2) Créer un CronJob

1. Choisir une image conteneur qui contient vos dépendances (ou votre appli). Privilégiez des tags immuables (ex. `:1.36` plutôt que `:latest`).
2. Définir l’horaire `spec.schedule` (« crontab-like »).
3. Empêcher les doublons avec `spec.concurrencyPolicy: Forbid` (ou `Replace`).
4. Régler l’historique et les réessais: `successfulJobsHistoryLimit`, `failedJobsHistoryLimit`, `jobTemplate.spec.backoffLimit`.
5. Gérer le cas « rattrapage » via `spec.startingDeadlineSeconds`.
6. Définir `resources` (limites/requests), variables d’environnement, ConfigMap/Secret, et un `serviceAccountName` si besoin de permissions.

Exemple un peu plus complet :

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

---

## 3) Options importantes à connaître

- schedule : expression cron standard.
- timeZone (≥1.27) : fuseau IANA (ex: `Europe/Paris`). Facilite les changements d’heure.
- concurrencyPolicy : `Allow` (par défaut), `Forbid` (n’autorise pas un nouveau job si le précédent est encore en cours), `Replace` (remplace le précédent).
- startingDeadlineSeconds : délai maximum pour rattraper une exécution manquée.
- suspend : met le CronJob en pause (stoppe les planifications futures ; n’arrête pas un job déjà démarré).
- successfulJobsHistoryLimit / failedJobsHistoryLimit : nombre de Jobs conservés.
- jobTemplate.spec.backoffLimit : nombre de réessais si le Pod échoue.
- jobTemplate.spec.activeDeadlineSeconds : tue le Job au-delà d’une durée.

---

## 4) Exemples utiles

- Toutes les 5 minutes, sans chevauchement :

```yaml
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
```

- Le lundi à 09:00, remplacement du job en cours si retard :

```yaml
spec:
  schedule: "0 9 * * 1"
  concurrencyPolicy: Replace
  startingDeadlineSeconds: 300
```

- Job Python avec venv embarqué dans l’image :

```yaml
containers:
  - name: job
    image: ghcr.io/monorg/worker:2.1.0
    args: ["python", "-m", "app.jobs.recalcule"]
```

- Export JSON et traitement en CLI (pratique avec jq) :

```bash
kubectl get cronjob -o json | jq '.items[] | {name: .metadata.name, schedule: .spec.schedule}'
```

> Astuce : pour manipuler proprement du JSON en CLI, voyez : [Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %}).

---

## 5) Débogage et observabilité

- État et événements :
  - `kubectl describe cronjob <name>`
  - `kubectl get jobs --selector=job-name=<prefix>` et `kubectl describe job/<name>`
  - `kubectl get events -A | grep -i cronjob`
- Pods et logs :
  - `kubectl get pods --selector=job-name=<job>`
  - `kubectl logs <pod>` (ou `-c <container>` si plusieurs conteneurs)
- Champs utiles :
  - `.status.lastScheduleTime` du CronJob
  - `.spec.startingDeadlineSeconds` et la politique de concurrence
- Problèmes fréquents :
  - Image introuvable, Secret/ConfigMap manquant → événements « Failed to pull image »/« not found ».
  - Job qui n’en finit pas → ajuster `activeDeadlineSeconds` et `concurrencyPolicy`.
  - Trop d’objets accumulés → baisser `successfulJobsHistoryLimit`/`failedJobsHistoryLimit` et mettre un TTLController (pour les Jobs si activé via `ttlSecondsAfterFinished`).

---

## 6) Migrer depuis cron (Linux) vers CronJob (K8s)

- Chemins et dépendances : tout doit exister dans l’image (ou via volumes). Pas de `/usr/local/bin` de l’hôte.
- Environnement : variables à définir via `env`, ConfigMap/Secret. Pas de `$HOME`/`PATH` implicite comme dans une session interactive.
- Droits : utilisez un `ServiceAccount` + RBAC adaptés. Évitez `root` si possible (SecurityContext).
- Fichiers et persistance : écrivez dans un volume (`emptyDir`, `PersistentVolumeClaim`, `configMap`/`secret` en lecture seule), pas dans le système de fichiers éphémère du Pod si vous devez conserver les résultats.
- Journalisation : stdout/stderr suffisent le plus souvent. Pour du centralisé, branchez Fluent Bit/Promtail vers Loki/ELK.

> Rappel syntaxe cron: consultez notre guide Linux ici: [Programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %}).

---

## 7) Bonnes pratiques

- Images immuables et petites (Alpine, distroless). Évitez `:latest`.
- `concurrencyPolicy: Forbid` pour les jobs non idempotents ; `Replace` pour les jobs rapides qu’on préfère relancer.
- Limitez mémoire/CPU et fixez des deadlines pour éviter les runaway jobs.
- Externalisez la config (ConfigMap/Secret) et ne logguez jamais des secrets.
- Surveillez l’historique et les événements ; alertez sur les pods en `CrashLoopBackOff` ou les jobs en échec.
- Si votre cluster ne tourne pas 24/7 (edge), utilisez `startingDeadlineSeconds` pour rattraper les exécutions manquées.

---

## FAQ

- Puis-je forcer l’exécution immédiate ?
  - Oui : créez un Job « one-shot » à partir du `jobTemplate` de votre CronJob ou utilisez `kubectl create job --from=cronjob/<name> <job-manuel>`.
- Comment arrêter temporairement une planification ?
  - Passez `spec.suspend: true` ; remettez à `false` pour reprendre.
- Comment éviter les doublons ?
  - `concurrencyPolicy: Forbid` et assurez-vous d’avoir des deadlines raisonnables.
- Fuseau horaire ?
  - Utilisez `spec.timeZone` si votre version de K8s le supporte (≥ 1.27). Sinon, l’horloge du contrôleur fait foi.

---

## Voir aussi

- [Linux : Programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %})
- [K8s : Comment déployer un cluster kubernetes bare-metal]({% post_url 2021-01-27-Comment-déployer-Kubernetes %})
- [Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %})

## Références

- [Documentation Kubernetes — CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [API reference Job/CronJob](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/cron-job-v1/)
- [Bonnes pratiques Jobs & CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
