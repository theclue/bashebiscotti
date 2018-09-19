---
layout: post
section-type: post
title: Initial setup
category: DEVOps
tags: [ 'kubernetes', 'docker', 'google cloud platform', 'data analysis', 'devops' ]
---

[Kubernetes](https://kubernetes.io) è una piattaforma gratuita e _open source_ per la gestione, l'orchestrazione e la scalabilità dei processi eseguiti in container [Docker](https://www.docker.com/) su dei cluster, spesso, ma non necessariamente, ospitati in una cloud.

Poiché già ho in produzione diversi carichi di lavoro in R che sono [eseguiti in parallelo](http://tamaszilagyi.com/blog/parallelizing-r-code-on-kubernetes/) sul mio cluster in esecuzione sulla Google Cloud Platform e che formano l'ossatura dei miei processi di [ETL](https://it.wikipedia.org/wiki/Extract,_transform,_load), ho pensato che fosse una buona idea portare sul cluster anche RStudio Server, l'IDE di sviluppo per lo sviluppo di processi in R, attualmente ospitato in una macchina virtuale Google Compute Engine. Questo, per diversi motivi:

Questo tutorial darà per scontate alcune conoscenze di base su Docker e Kubernetes e si occuperà solo del deploy, appunto, di RStudio. Qualora fosse necessaria una introduzione a Kubernetes, vi rimando al paragrafo dei riferimenti.

Il deploy verrà effettuato su Google Cloud Platform, ma la procedura è valida - con poche modifiche - anche su altre piattaforme come OpenShift, DigitalOcean o [Minikube](https://kubernetes.io/docs/setup/minikube/).

### Riferimenti

Le seguenti risorse sono state consultate per la stesura dell'articolo. Ad esse rimando per un approfondimento più dettagliato sui temi specifici.

* [R on Kubernetes - serverless Shiny, R APIs and scheduled scripts](http://code.markedmondson.me/r-on-kubernetes-serverless-shiny-r-apis-and-scheduled-scripts/) dal blog di Mark Edmondson
* [Setting up HTTP access with Ingress](https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer)
* [Ingress with nginx](https://cloud.google.com/community/tutorials/nginx-ingress-gke)
* [Documentazione ufficialer di Kubernetes](https://kubernetes.io/docs/reference/kubectl/overview/)
* StackOverflow in particolare su [questo](https://stackoverflow.com/questions/52303064/configuring-rstudio-server-service-for-nginx-ingress-in-gke/52364023#52364023) e [questo](https://stackoverflow.com/questions/48452556/setting-up-a-kuberentes-cluster-with-http-load-balancing-ingress-for-rstudio-and) argomento

### Configurazione iniziale

Per prima cosa, creiamo un cluster su Google Kubernetes Engine. Effettueremo il tutto tramite i comandi da shell della [Cloud SDK](https://cloud.google.com/sdk/). Dovremo solo aggiungere un paio di componenti opzionali, non installati per default, che useremo per questo tutorial.

<pre><code data-trim class="bash">
# Client per interagire con il cluster Cubernetes
gcloud components install kubectl

# Gestore che abilita il caricamento delle immagini Docker su Google Cloud Registry privato di progetto
gcloud components install docker-credential-gcr
</code></pre>
