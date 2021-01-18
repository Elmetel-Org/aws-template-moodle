## template-moodle
18/01/2021
- [reference-official](#reference-official)
- [reference](#reference)
- [repository](#repository)
- [Hosting-Moodle-su-AWS](#Hosting-Moodle-su-AWS)
  - [repository](#repository)
  - [Panoramica](#panoramica)
  - [In-dettaglio](#in-dettaglio)
    - [Architettura](#architettura)
    - [AWS-Certificate-Manager](#aws-certificate-manager)
    - [Application-Load-Balancer](#application-load-balancer)
    - [Amazon-Autoscaling](#amazon-autoscaling)
    - [Amazon-Elastic-File-System-(EFS)](#amazon-elastic-file-system-efs)
    - [Caching](#caching)
      - [OPcache](#opcache)
      - [Amazon-ElastiCache](#amazon-elasticache)
	  - [Session-Caching](#session-caching)
      - [Application-Caching](#application-caching)
      - [Amazon-CloudFront](#amazon-cloudfront)
    - [Amazon-Route-53](#amazon-route-53)

---
### repository
questo repository è un fork del repository:
https://github.com/aws-samples/aws-refarch-moodle

---
### installazione
Per fare l'installazione bisogna cliccare nel file README.md
che si trova nel repository

- in `EC2 Key Pair` selezionare `moodle`
- in `Use Route 53` selezionare `false`
- in `VpcId` selezionare `vpc-6b2bae01 (172.31.0.0/16)`
- in `Public Subnets` selezionare tutte le opzioni
- in `Private Subnets` selezionare tutte le opzioni
- in `Add dummy data (GiB)` aggiungere un valore maggiore di 0. es: 1000
- in `Web ASG Max` settare il numero massimo di istanze `EC2`, es: 2
- in `Web ASG Min` settare il numero minimo di istanze `EC2`, es: 1
- in `AvailabilityZones` selezionare tutte le opzioni

---
## Hosting-Moodle-su-AWS
Version 1.0.0

---
### Panoramica

Questo repository è costituito da un set di modelli nidificati che distribuiscono un ambiente Moodle altamente disponibile, elastico e scalabile su AWS.

Questa architettura di riferimento fornisce un set di modelli YAML per la distribuzione di Moodle su AWS utilizzando:

- [Amazon Virtual Private Cloud (Amazon VPC)](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Introduction.html)
- [Amazon Elastic Compute Cloud (Amazon EC2)](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)
- [Auto Scaling](http://docs.aws.amazon.com/autoscaling/latest/userguide/WhatIsAutoScaling.html)
- [Elastic Load Balancing (Application Load Balancer)](http://docs.aws.amazon.com/elasticbalancing/latest/application/introduction.html)
- [Amazon Relational Database Service (Amazon RDS)](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html)
- [Amazon ElastiCache](http://docs.aws.amazon.com/AmazonElastiCache/latest/UserGuide/WhatIs.html)
- [Amazon Elastic File System (Amazon EFS)](http://docs.aws.amazon.com/efs/latest/ug/whatisefs.html)
- [Amazon CloudFront](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)
- [Amazon Route 53](http://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Amazon Certificate Manager (Amazon ACM)](http://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)
- with [AWS CloudFormation](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)

Questa architettura può essere eccessiva per molte distribuzioni di Moodle, tuttavia i modelli possono essere eseguiti individualmente e / o modificati per distribuire un sottoinsieme dell'architettura che si adatta alle tue esigenze.

---
### In-dettaglio
Se vuoi solo distribuire lo stack Moodle, segui questi passaggi. Puoi leggere i dettagli di seguito per comprendere meglio l'architettura.

1. Se prevedi di utilizzare `TLS`, devi creare o importare il certificato in Amazon Certificate Manager prima di avviare Moodle.
2. Distribuisci lo stack 00-master.yaml. **Non abilitare il caching della sessione in ElastiCache e lasciare entrambe le dimensioni Min e Max Auto Scaling Group (ASG) impostate su uno.** La procedura guidata di installazione non verrà completata se la cache della sessione è configurata.
3. Una volta completata la distribuzione dello stack, accedere al sito Web per completare l'installazione di Moodle. _NOTE: È possibile che si verifichi un errore 504 Gateway Timeout o CloudFront nel passaggio finale della procedura guidata di installazione (dopo aver impostato la password dell'amministratore). Puoi semplicemente aggiornare la pagina per completare l'installazione._ Potresti anche vedere "L'installazione deve essere completata dall'indirizzo IP originale, mi dispiace". per risolvere questo problema dovrai aggiornare il tuo database e impostare il campo lastip della tabella mdl_user sull'indirizzo IP interno del tuo ALB (puoi trovarlo guardando le interfacce di rete dalla pagina EC2 nella Console AWS). Dal webserver puoi eseguire:
   psql -h <hostname> -U<Username>
   update mdl_user set lastip='<ip address>';
4. Configurare la memorizzazione nella cache dell'applicazione nella configurazione del sito Moodle (vedere di seguito per i dettagli).
5. Ora puoi aggiornare lo stack appena distribuito per abilitare la memorizzazione nella cache della sessione e impostare i valori di dimensione del gruppo Auto Scaling Min e Max come desiderato.

\*Note: puoi raggiungere il server web modificando il numero minimo di bastion host su 1 nel gruppo Auto Scaling o abilitare Systems Manager Session Manager aggiornando il ruolo IAM assegnato all'istanza del server web (https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started-instance-profile.html)

Puoi avviare questo stack CloudFormation, utilizzando il tuo account, nelle seguenti regioni AWS.
Il modello funzionerà in altre regioni poiché Aurora PostgreSQL viene distribuito a livello globale.


| AWS Region Code | Name | Launch |
| --- | --- | --- 
| us-east-1 |US East (N. Virginia)| [![cloudformation-launch-stack](images/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=Moodle&templateURL=https://s3.amazonaws.com/aws-refarch/moodle/latest/templates/00-master.yaml) |
| us-east-2 |US East (Ohio)| [![cloudformation-launch-stack](images/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/new?stackName=Moodle&templateURL=https://s3.amazonaws.com/aws-refarch/moodle/latest/templates/00-master.yaml) |
| us-west-2 |US West (Oregon)| [![cloudformation-launch-stack](images/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=Moodle&templateURL=https://s3.amazonaws.com/aws-refarch/moodle/latest/templates/00-master.yaml) |
| eu-west-1 |EU (Ireland)| [![cloudformation-launch-stack](images/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/new?stackName=Moodle&templateURL=https://s3.amazonaws.com/aws-refarch/moodle/latest/templates/00-master.yaml) |
| eu-central-1 |EU (Frankfurt)| [![cloudformation-launch-stack](images/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-central-1#/stacks/new?stackName=Moodle&templateURL=https://s3.amazonaws.com/aws-refarch/moodle/latest/templates/00-master.yaml) |
| ap-southeast-2 |AP (Sydney)| [![cloudformation-launch-stack](images/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-2#/stacks/new?stackName=Moodle&templateURL=https://s3.amazonaws.com/aws-refarch/moodle/latest/templates/00-master.yaml) |

---
### Architettura
Le sezioni seguenti descrivono i singoli componenti dell'architettura.

![](images/aws-refarch-moodle-architecture.png)

---
### AWS-Certificate-Manager
AWS Certificate Manager è un servizio che ti consente di fornire, 
gestire e distribuire facilmente certificati Secure Sockets Layer/Transport Layer Security (SSL/TLS) da utilizzare con i servizi AWS. 
È necessario eseguire SSL/TLS per proteggere sessioni e password. 
Se prevedi di utilizzare Transport Layer Security (TLS), devi creare o importare un certificato TLS in AWS Certificate Manager prima di avviare il modello. 
Inoltre, se si utilizza CloudFront e si ospita Moodle in una regione diversa da us-east-1, è necessario creare o importare il certificato sia in us-east-1 che nella regione in cui si ospita Moodle. 
CloudFront utilizza sempre i certificati da us-east-1.

---
### Application-Load-Balancer
Un Application Load Balancer distribuisce il traffico delle applicazioni in entrata su più istanze EC2 in più zone di disponibilità. 
Ottieni un'elevata disponibilità raggruppando più server Moodle dietro un bilanciatore del carico. 
Moodle fornisce una panoramica di [Server Clustering](https://docs.moodle.org/34/en/Server_cluster) che dovresti esaminare prima di procedere. 
Il modello avvierà un Application Load Balancer (ALB).

---
### Amazon-Autoscaling
Amazon EC2 Auto Scaling ti aiuta a garantire di avere il numero corretto di istanze Amazon EC2 disponibili per gestire il carico della tua applicazione. 
Il modello configura la scalabilità automatica in base all'utilizzo della CPU.
Viene aggiunta un'istanza aggiuntiva quando l'utilizzo medio della CPU supera il 75% per tre minuti e rimossa quando l'utilizzo medio della CPU è inferiore al 25% per tre minuti. 
In base al tipo di istanza in esecuzione, alla configurazione della cache scelta e ad altri fattori, potresti scoprire che un'altra metrica è un migliore predittore di carico. 
Sentiti libero di modificare le metriche secondo necessità.

*Nota che la procedura guidata di installazione causa picchi nella CPU che potrebbero causare una scalabilità imprevista del cluster. 
Inizialmente, dovresti distribuire un singolo server impostando la dimensione ASG minima e massima su uno. 
Quindi, completa la procedura guidata di configurazione di Moodle e aggiorna lo stack di CloudFormation impostando la dimensione ASG minima e massima sui limiti desiderati.*

---
### Amazon-Elastic-File-System-(EFS)
Amazon Elastic File System (Amazon EFS) fornisce uno storage di file semplice e scalabile da utilizzare con le istanze Amazon EC2 nel cloud AWS. 
Mentre l'installazione di Moodle su EFS semplifica la gestione (aggiornamenti, patch, ecc.), Moodle non funziona bene quando viene eseguito dallo storage condiviso. 
Moodle consiglia di configurare dirroot sulla memoria locale.
Pertanto, il modello utilizza una combinazione di EBS (Elastic Block Storage) e di archiviazione EFS. 
Ciascun server Web nel cluster Moodle utilizza la seguente struttura di directory:

```
$CFG->dirroot = '/var/www/moodle/html'        #Stored on root EBS volume
$CFG->dataroot = '/var/www/moodle/data'       #Stored on shared EFS filesystem
$CFG->cachedir = '/var/www/moodle/cache'      #Stored on shared EFS filesystem
$CFG->tempdir = '/var/www/moodle/temp'        #Stored on shared EFS filesystem
$CFG->localcachedir = '/var/www/moodle/local' #Stored on root EBS volume
```

Il throughput di Elastic File System scala con la crescita del file system. Inizialmente EFS disporrà di uno spazio di archiviazione minimo. 
Pertanto, il modello offre la possibilità di eseguire il seeding del file system con i dati per stabilire le prestazioni di base. Inoltre, è possibile monitorare le prestazioni del file system e aggiungere dati aggiuntivi se necessario. 
C'è una discussione approfondita di questo nell'architettura di riferimento di WordPress. 
Tuttavia, poiché abbiamo scelto di installare Moodle sullo storage locale (l'architettura di riferimento di WordPress installata su EFS), questo dovrebbe essere meno preoccupante di quanto lo fosse per WordPress.

*Moodle consiglia quando si eseguono soluzioni in cluster
[dirroot dovrebbe essere sempre letto solo per il processo apache perché altrimenti l'installazione e la disinstallazione del plugin incorporate porterebbero i nodi fuori sincrono](https://docs.moodle.org/34/en/Server_cluster#.24CFG-.3Edirroot)
Come implica questa dichiarazione, non è possibile installare plug-in su un cluster di server dalla pagina di amministrazione. 
Moodle consiglia di installare manualmente i plugin su ciascun server durante la manutenzione pianificata. 
Per rimanere fedeli alla metodologia dell'infrastruttura come codice utilizzata da CloudFormation, dovremmo creare uno script per l'installazione dei plugin. 
In alternativa, puoi installare il plug-in su un singolo server, creare un'AMI e aggiornare la configurazione di avvio.*

---
### Caching
La memorizzazione nella cache può avere un impatto notevole sulle prestazioni di Moodle. Il modello configurerà varie forme di memorizzazione nella cache tra cui OPcache, CloudFront ed ElastiCache.

---
#### OPcache
Opcache accelera l'esecuzione di PHP memorizzando nella cache gli script precompilati. I vantaggi di OPcache sono prestazioni migliorate e un utilizzo della memoria notevolmente inferiore. Il modello configura OPcache come descritto [qui](https://docs.moodle.org/34/en/OPcache).

---
#### Amazon-ElastiCache
Amazon ElastiCache per Memcached è un servizio di archivio di valori-chiave in memoria compatibile con Memcached che può essere utilizzato come cache o archivio dati. 
Moodle consiglia di 
[non utilizzare lo stesso server memcached per entrambe le sessioni e MUC. Gli eventi che attivano l'eliminazione delle cache MUC portano all'eliminazione di MUC dal server memcached](https://docs.moodle.org/26/en/Session_handling).
Pertanto, il modello configura due cluster elasticache, uno per la memorizzazione nella cache della sessione e uno per la memorizzazione nella cache dell'applicazione.

---
#### Session-Caching
Moodle consiglia di 
[memorizzare le sessioni utente in un server memcached condiviso](https://docs.moodle.org/34/en/Server_cluster#Performance_recommendations)
Il modello configura la memorizzazione nella cache della sessione come descritto [qui] (https://docs.moodle.org/34/en/Session_handling#Memcached_session_driver).

*Note: l'installazione guidata di Moodle fallisce se la memorizzazione nella cache della sessione memcached è abilitata durante la configurazione iniziale. Pertanto, è necessario lasciare la cache della sessione disabilitata durante l'installazione iniziale, quindi aggiornare il modello per abilitare la cache della sessione dopo aver completato l'installazione.*

---
#### Application-Caching
Il modello distribuisce un cluster ElastiCache per la memorizzazione nella cache dell'applicazione, ma la memorizzazione nella cache dell'applicazione deve essere configurata dopo l'avvio. 
È possibile configurare memcache come descritto 
[qui](https://docs.moodle.org/28/en/Caching#Memcached) inserendo l'endpoint di rilevamento automatico nell'elenco dei server sia in Configurazione archivio che in Abilita server cluster (vedere immagine sotto). 
È possibile trovare l'indirizzo dell'endpoint negli output dello stack di memorizzazione nella cache dell'applicazione. Infine, scorri fino alla fine della pagina di amministrazione della cache in Moodle e imposta elasticache come archivio predefinito per la memorizzazione nella cache dell'applicazione.
![](images/aws-refarch-moodle-caching.png)

#### Amazon CloudFront 

Amazon CloudFront is a global content delivery network (CDN) service that securely delivers data, videos, applications, and APIs to your viewers with low latency and high transfer speeds. The template can optionally configure CloudFront to cache content closer to your users. This is especially beneficial if your users are spread across a large geographic area. For example, remote students in an online program.

### Amazon Route 53

Amazon Route 53 is a highly available and scalable cloud Domain Name System (DNS) web service. The template will optionally configure a Route53 alias that points to either the Application Load Balancer or CloudFront. If you are using another DNS system, you should create a CNAME record in your DNS system to reference either the Application Load Balancer or CloudFront (if deployed). If you don't have access to DNS you can leave Domain Name blank and the template will configure Moodle to use the auto-generated Application Load Balancer domain name. 

## License

This library is licensed under the Apache 2.0 License. 

Portions copyright.

- Moodle is licensed under the General Public License (GPLv3 or later) from the Free Software Foundation.
- OPcache is licensed under PHP License, version 3.01.

Please see LICENSE for applicable license terms and NOTICE for applicable notices.
