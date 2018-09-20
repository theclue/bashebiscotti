---
layout: post
section-type: post
title: RStudio Server su Google Kubernetes Engine
excerpt: "RStudio è un popolare ambiente di sviluppo per il linguaggio R. In questo articolo vedremo come utilizzarlo in un cluster Kubernetes."
category: DEVOps
tags: [ 'kubernetes', 'docker', 'google cloud platform', 'data analysis', 'devops' ]
image:
  path: /assets/immagini/rstudio-kubernetes-header.jpg
  thumbnail: /assets/immagini/rstudio-kubernetes-thumb.jpg
  caption: "Photo from [WeGraphics](http://wegraphics.net/downloads/free-ultimate-blurred-background-pack/)"
---

[Kubernetes](https://kubernetes.io) è una piattaforma gratuita e _open source_ per la gestione, l'orchestrazione e la scalabilità dei processi eseguiti in container [Docker](https://www.docker.com/) su dei cluster, spesso, ma non necessariamente, ospitati in una cloud.

Poiché già ho in produzione diversi carichi di lavoro in R che sono [eseguiti in parallelo](http://tamaszilagyi.com/blog/parallelizing-r-code-on-kubernetes/) sul mio cluster in esecuzione sulla Google Cloud Platform e che formano l'ossatura dei miei processi di ETL e di Machine Learning, ho pensato che fosse una buona idea portare sul cluster anche RStudio Server, l'IDE di sviluppo per lo sviluppo di processi in R, prima di oggi ospitato in una macchina virtuale Google Compute Engine.

Questo tutorial darà per scontate alcune conoscenze di base su [Docker](https://www.docker.com/) e [Kubernetes](https://kubernetes.io/) e si occuperà solo del deploy, appunto, di RStudio Server, la versione di RStudio dotata di interfaccia web. Qualora fosse necessaria una introduzione a Kubernetes, vi rimando al paragrafo dei riferimenti dove ne è riportata la guida ufficiale.

Il deploy verrà effettuato su Google Cloud Platform, ma la procedura è valida - con poche modifiche - anche su altre piattaforme come OpenShift, DigitalOcean o [Minikube](https://kubernetes.io/docs/setup/minikube/).

### Riferimenti

Le seguenti risorse sono state consultate per la stesura dell'articolo. Vi rimando ad esse per un approfondimento più dettagliato sui temi specifici.

* [R on Kubernetes - serverless Shiny, R APIs and scheduled scripts](http://code.markedmondson.me/r-on-kubernetes-serverless-shiny-r-apis-and-scheduled-scripts/) di [Mark Edmondson](http://markedmondson.me/)
* [Setting up HTTP access with Ingress](https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer)
* [Ingress with nginx](https://cloud.google.com/community/tutorials/nginx-ingress-gke)
* [Documentazione ufficiale di Kubernetes](https://kubernetes.io/docs/reference/kubectl/overview/)
* StackOverflow, in particolare le risposte a  [questa](https://stackoverflow.com/questions/52303064/configuring-rstudio-server-service-for-nginx-ingress-in-gke/52364023#52364023) e [questa](https://stackoverflow.com/questions/48452556/setting-up-a-kuberentes-cluster-with-http-load-balancing-ingress-for-rstudio-and) domanda.

### Perché RStudio su Kubernetes?

Ci sono diversi motivi per cui ha senso containerizzare ed eseguire RStudio in un cluster Kubernetes.

* __Ottimizzazione delle risorse del cluster__: sfruttare il cluster per eseguire dei servizi stateful interattivi è una cosa con cui non tutti sono d'accordo, ma l'alternativa è dover disporre di una istanza di una virtual machine (come Google Cloud Engine) apposita su cui installare il server. Se si dispone già di un cluster in produzione, questi soldi possono essere risparmiati.
* __Facilità di installazione e di gestione__: non dover installare e gestire manualmente un server e assicurare coerenza tra gli ambienti di sviluppo e produzione è il motivo principale per cui Docker è stato inventato. Portare un servizio dockerizzato su Kubernetes è solo l'ultimo traguardo.
* __Coerenza di risorse tra processi batch e applicativi__: portare il server di sviluppo sullo stesso cluster in cui sono, per ipotesi, in esecuzione i processi R batch di produzione significa che questi potranno accedere alle stesse risorse del cluster: CPU, RAM, dischi condivisi, firewall, connettività di rete…
* __Scalabilità__: RStudio Server nella sua versione gratuita non è esattamente uno strumento facile da scalare su un cluster, ma anche con una sola replica, Kubernetes offre il vantaggio di automatizzarne l'orchestrazione e di garantire uptime, restart e controllo dello stato di salute dei servizi, intervenendo in autonomia quando necessario.

### Configurazione iniziale e creazione del cluster

Per prima cosa, creiamo un cluster su Google Kubernetes Engine. Effettueremo il tutto tramite i comandi da shell della [Cloud SDK](https://cloud.google.com/sdk/), che vi invito ad installare. C'è, però, da aggiungere un paio di componenti opzionali, non installati per default, che useremo per questo tutorial. Uno di questi è [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/), il client a riga di comando essenziale per creare e gestire un cluster.

Poi, effettueremo alcuni settaggi di default sul progetto, per risparmiare tempo nei comandi successivi.

```sh
# Può essere installato direttamente da gcloud
gcloud components install kubectl

# Settaggi di default sul progetto
#
# Per avere la lista dei progetti: gcloud projects list
# Le lista delle region disponibili https://cloud.google.com/compute/docs/regions-zones/
gcloud config set project <PROJECT_ID>
gcloud config set compute/zone <REGION>

# Creazione di un cluster composto da 3 nodi
gcloud container clusters create r-cluster --num-nodes=3

# Autenticazione sul cluster r-cluster: permette di usare il comando kubectl
gcloud container clusters get-credentials r-cluster
```

### Setup di Nginx Ingress Controller

Per impostazione predefinita, i servizi in esecuzione sui nodi del cluster non sono accessibili all'esterno del cluster. Questo ha totalmente senso, non solo per la sicurezza, ma anche perché essendo applicazioni pensate per essere scalate su più nodi, in base al carico macchina degli stessi, non è possibile sapere a priori quale nodo è in grado di assolvere alle richieste.

Quello che va fatto, invece, è di depositare sul cluster un servizio di __ingress__ che si occupi del __mapping__ dei servizi sul server e del __routing__ ai nodi, secondo policy di carico, scalabilità, eccetera.

Google mette a disposizione un modulo di ingress predefinito, ma che però non è molto indicato per applicazioni dotate di interfaccia web o che comunicano tramite protocollo http(s) (per esempio servizi di API REST). Molto più indicato, per questo scopo è [Nginx Ingress](https://kubernetes.github.io/ingress-nginx/), perché consente di mappare i processi in esecuzione sul cluster in endpoint web, come ad esempio `app1`, `/app2`, ecc.

L'installazione è molto semplice se eseguita tramite [Helm](https://helm.sh/), il gestore di pacchetti di Kubernetes.

Non c'è nessun motivo per installare ancora Nginx Ingress manualmente a meno di non dover alterare pesantemente il file `nginx.conf` o per motivi didattici. Sappiate, però, che [la procedura è molto più lunga](https://hackernoon.com/setting-up-nginx-ingress-on-kubernetes-2b733d8d2f45).

La prima cosa da fare è installare Helm sulla propria macchina locale. Lo script sotto è in flavour Unix. Su Windows, invece, [l'installazione è un po' meno elegante](https://medium.com/@JockDaRock/take-the-helm-with-kubernetes-on-windows-c2cd4373104b), ma ancora funziona.

```sh
curl -o get_helm.sh https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get
chmod +x get_helm.sh
./get_helm.sh
```

A questo punto è necessario installare Tiller sul cluster, l'agente che effettivamente farà il deploy su Kubernetes dei pacchetti preparati dal Helm. Per prima cosa creiamo un service account per Tiller.

```sh
kubectl create serviceaccount tiller --namespace kube-system
```

E diamogli il ruolo di _cluster admin_, in modo che abbia i privilegi per deployare servizi sul cluster. Il deployment file yaml per la configurazione è il seguente:

{% gist 8d7adce32ae57d5a6baccae309d0d0c7 rstudio-kubernetes.tiller.yaml %}

Lo prendiamo direttamente dal gist di Github in cui l'ho reso disponibile e lo applichiamo sul cluster. La possibilità che ci da `kubectl` di applicare i deployment file da una sorgente remota è uno di quei piccoli tocchi di classe che fanno risparmiare tempo e rendono il lavoro meno noioso.

```sh
kubectl create -f https://gist.github.com/theclue/8d7adce32ae57d5a6baccae309d0d0c7/raw/7b630f9cf67824d9d6728f1f8ef33248470ea9ea/rstudio-kubernetes.tiller.yaml
```

A questo punto, basta installare Tiller e verificare che funzioni. Chiaramente, `helm ls` non torna alcun risultato perché non ci sono pacchetti installati, ma basta che il comando non torni errori.

```sh
helm init --service-account tiller --upgrade

# Dopo qualche secondo, verificare che questo comando non dia errore
helm ls
```

### Deploy di Nginx-ingress

Con Helm, l'operazione è praticamente immediata:

```sh
helm install --name nginx-ingress stable/nginx-ingress --set rbac.create=true
```

Possiamo verificare che il controller sia stato depositato verificando i servizi disponibili sul cluster:

```sh
kubectl get service -o wide

NAME                            TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
kubernetes                      ClusterIP      10.7.240.1     <none>           443/TCP
nginx-ingress-controller        LoadBalancer   10.7.250.88    ***.***.***.***  80:31111/TCP,443:31961/TCP
nginx-ingress-default-backend   ClusterIP      10.7.244.103   <none>           80/TCP
```

Notate come vi sia un indirizzo IP esterno assegnato al servizio (qui mascherato, nel  mio output): è l'indirizzo del Load Balancer, esposto all'esterno e a cui indirizzare le richieste http(s), sulle porte 80 e 433. Il servizio di _default backend_, non accessibile all'esterno (ma solo da e mediante il Load Balancer) è un semplice web server con il solo scopo di mostrare una pagina 404 per qualunque endpoint non censito nelle tabelle di routing dell'ingress. Verifichiamone al volo il funzionamento:

```sh
$ curl ***.***.***.***/scemochilegge
default backend - 404
```

La pagina 404 predefinita può essere sostituita [per esempio con questa](https://github.com/theclue/default-backend) sostituendo il componente di _default backend_ con uno customizzato durante l'installazione tramite Helm.

Tanto per essere ancora più chiari: sì, solo il Load Balancer ha un IP pubblico. Tutti i servizi in esecuzione sul server passano attraverso di esso in base alle sue regole di routing. Regole che andremo a impostare tra qualche minuto, contestualmente al deploy di RStudio.

### Deploy di RStudio server

A questo punto manca il deploy di RStudio Server. Operazione resa estremamente semplice dal fatto che un'immagine Docker a questo scopo esiste in quanto parte del progetto [Rocker](https://www.rocker-project.org/). C'è solo da deployarla sul server.

È plausibile che vogliate crearvi un dockerfile customizzato per le vostre esigenze partendo da questa immagine e aggiungendo pacchetti di sistema o package R che sapete che andrete ad utilizzare spesso, per poi caricare la vostra immagine personalizzata sul vostro spazio [DockerHub](https://hub.docker.com/) o sul Google Cloud Private Registry. In questo tutorial ci atterremo, tuttavia, allo scenario di base.

Utilizzando uno dei dockerfile di base l'operazione è molto semplice:

```sh
kubectl run rstudio --image rocker/rstudio:latest --port 8787 --env="PASSWORD=<tuapassword>" -replica=1
kubectl expose deployment rstudio --target-port=8787 --port=8787 --type=NodePort
```

Il deploy è esposto come NodePort, che ci assicura che la porta 8787 sia aperta al Load Balancer in tutti i nodi. Questo è, per la verità, superfluo, non avendo deciso nello specifico di effettuare un port forwarding. Lo stesso avremmo potuto ottenerlo esponendo un servizio di tipo ClusterIP.

A titolo dimostrativo, qui ci atteniamo alla modalità di base, e meno sicura, per impostare la password iniziale che avrà l'utente `rstudio`, ovvero attraverso la definizione di una variabile d'ambiente in chiaro. Molto meglio sarebbe utilizzare, per il medesimo scopo, i [secrets](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/) di Kubernetes.

Osserviamo che il deploy sia avvenuto correttamente:

```sh
kubectl get service -o wide

NAME                            TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
kubernetes                      ClusterIP      10.7.240.1     <none>           443/TCP
nginx-ingress-controller        LoadBalancer   10.7.250.88    ***.***.***.***  80:31111/TCP,443:31961/TCP
nginx-ingress-default-backend   ClusterIP      10.7.244.103   <none>           80/TCP
rstudio                         NodePort       10.7.243.134   <none>           80:31703/TCP
```

Per finire, dettiamo la regola di Ingress affinché il servizio sia accessibile dall'endpoint `<EXTERNAL_IP>/rstudio`. Lo facciamo attraverso una risorsa di tipo _ingress_, ovvero molto banalmente una regola di routing che mappa l'endpoint `/rstudio/` al deploy `rstudio`.

{% gist 8d7adce32ae57d5a6baccae309d0d0c7 rstudio-kubernetes.rstudio-ingress.yaml %}

Qui le _annotation_ hanno un ruolo importante:

* Istruiscono Kubernetes ad utilizzare nginx cone ingress
* Si assicurano che Nginx funga anche da reverse proxy. RStudio usa, infatti, dei redirect lato client che punterebbero a `/`e che vanno riscritti in modo da puntare a `/rstudio/`
* Assicurano che una sessione, individuata da un apposito cookie, non sia spezzettata tra nodi diversi. RStudio è una applicazione stateful. Per quanto per ora abbiamo optato per avere un solo pod, e quindi l'eventualità che nodi diversi rispondano alle richieste è esclusa, queste annotazioni ci semplificheranno il lavoro in futuro, quando renderemo il servizio scalabile.

```sh
kubectl create -f https://gist.github.com/theclue/8d7adce32ae57d5a6baccae309d0d0c7/raw/7b630f9cf67824d9d6728f1f8ef33248470ea9ea/rstudio-kubernetes.rstudio-ingress.yaml
```
Verifichiamo, stavolta dal browser, che il servizio funzioni. L'indirizzo IP da utilizzare è quello del Load Balancer. La username da utilizzare per la login è `rstudio` mentre la password è quella scelta durante il deploy dell'immagine Docker di RStudio.

![RStudio Server Login]({{ '/assets/immagini/rstudio-kubernetes-login.png' | absolute_url }})

### Prossimi passi

Con questi rapidi passaggi abbiamo reso disponibile un servizio RStudio Server sul nostro cluster e, contestualmente, abbiamo attivato un servizio di _Ingress_ che potremo utilizzare per gli altri servizi http che intendiamo pubblicare sul cluster. Possiamo andare, però, oltre e migliorare il setup, soprattutto in termini di sicurezza e scalabilità:

* Utilizzare Nginx per forzare le connessioni su un canale SSL. Tramite helm è possibile aggiungere certificati Let's Encypt [con facilità](https://medium.com/containerum/how-to-launch-nginx-ingress-and-cert-manager-in-kubernetes-55b182a80c8f).
* [Aggiungere un volume di storage persistente](https://cloud.google.com/kubernetes-engine/docs/tutorials/persistent-disk) accessibile dal servizio.
* Migliorare la sicurezza. La versione gratuita di RStudio Server, infatti, è molto carente da questo punto di vista, ma possiamo sfruttare Nginx per introdurre un livello di autenticazione, magari [di tipo OAuth2](https://blog.n1analytics.com/oauth2-lets-encrypt-and-k8s/).

Certo, l'ideale sarebbe di rendere realmente scalabile l'istanza, ma alcuni limiti di RStudio rendono la cosa veramente complessa, soprattutto per ciò che concerne la gestione degli utenti (non ha senso scalare l'istanza se solo l'utente `rstudio` può accedere al servizio). Qualcosa, però, si può tentare e ci tornerò in futuro. Nel frattempo, godetevi l'istanza di RStudio su Kubernetes e, vi prego, almeno il primo giorno…[non cercate di fare autenticazioni OOB](https://support.rstudio.com/hc/en-us/articles/217952868-Generating-OAuth-tokens-for-a-server-using-httr) su servizi che non lo supportano, come Facebook. Vi renderà tristi e vi farà odiare Facebook.
