<h1 align="center">VIRTUALIZZAZIONE</h1>

### 1. Aspetti Importanti nella Realizzazione di un VMM (Requisiti)

Il Virtual Machine Monitor (VMM, o Hypervisor) consente la condivisione da parte di più macchine virtuali della stessa piattaforma hardware. Gestisce le interazioni tra le macchine virtuali e l’hardware sottostante, in modo da garantire:

- isolamento tra le macchine virtuali;
- stabilità del sistema.

Il VMM deve quindi offrire le risorse virtuali necessarie per il funzionamento delle macchine virtuali, tra cui:
- CPU
- memoria (RAM)
- dispositivi di I/O.

A tal scopo sono necessari diversi requisiti per la realizzazione del VMM.

1. Ambiente di esecuzione dei programmi praticamente identico a quello della macchina reale: garantire che gli stessi programmi che funzionano su architetture non virtualizzate possano essere eseguiti nelle VM senza modifiche.

2. Garantire una elevata efficienza nell’esecuzione dei programmi: le istruzioni non privilegiate devono venire eseguite direttamente in hardware senza coinvolgere il VMM.

3. Garantire la sicurezza e la stabilità dell’intero sistema: i programmi in esecuzione sulle macchine virtuali non possono effettuare l’accesso in modo privilegiato all’hardware.

### 2. Compiti di un VMM

Il VMM è l'unico mediatore tra HW e SW, e ha il compito di consentire la condivisione di una singola macchina HW a più VM guest, realizzando per ciascuna una sandbox al fine di garantire isolamento e stabilità (del sistema e delle singole VM).
Nel caso di un VMM di sistema dev'essere l'unico componente ad avere il pieno controllo dell'HW e a poter eseguire le istruzioni privilegiate (unico componente ad eseguire al ring 0).
Ha il compito di gestione delle VM: creazione, spegnimento/accensione, eliminazione, migrazione live.

### 3. Classificazione del VMM

I diversi tipi di VMM si classificano in base a due parametri:
1. il *livello* a cui si collocano:
- **VMM di Sistema** - il VMM esegue <ins>direttamente sull'HW</ins> e consiste in un sistema operativo leggero che viene corredato dei driver per pilotare le varie periferiche. In assenza di multiboot è necessario disinstallare il sistema operativo preesistente.
- **VMM Ospitato** - il VMM è un'<ins>applicazione</ins> che esegue su un sistema operativo preinstallato, al pari delle altre applicazioni. Le singole VM guest sono anch'esse applicazioni.
2. la *modalità di dialogo* tra le VM guest ed il VMM per l'utilizzo dell'HW sottostante:
- **Virtualizzazione Pura** - le VM guest utilizzano la <ins>stessa insterfaccia</ins> (istruzioni macchina) <ins>fornita dall'architettura fisica</ins>. Generalmente è il caso di HW con supporto nativo alla virtualizzazione.
- **Paravirtualizzazione** - il VMM presenta alle VM guest un'interfaccia "virtuale", differente da quella fornita dall'HW (<ins>hypercall API</ins>). È una delle possibili soluzioni software che vengono adottate quando l'HW non fornisce supporto nativo alla virtualizzazione. È il caso di XEN.

### 4. Effettuare un confronto tra virtualizzazione di sistema e ospitata, con schema.

In base a dove è collocato il Virtual Machine Monitor si hanno due tipo di virtualizzazione:

- VMM di sistema;
- VMM ospitati.

Nel caso di **VMM di sistema** le funzionalità di virtualizzazione vengono integrate in un sistema operativo leggero (il VMM), posto direttamente sopra l’hardware dell’elaboratore. Per garantire un corretto funzionamento del VMM, occorre disporre di tutti i driver necessari per pilotare le periferiche.
Si hanno due macro componenti importanti:
- **Host**: piattaforma di base sulla quale si realizzano le macchine virtuali. Include la macchina fisica e il VMM;
- **Guest**: le macchine virtuali, che includono applicazioni e sistema operativo.
Esempi di VMM di sistema sono kvm, xen, VMware svsphere e Microsoft HyperV.

Nel caso invece di **VMM ospitato**, si installa il VMM come un’applicazione sopra un sistema operativo esistente. Il VMM opera nello spazio utente e accede all'hardware tramite system call del sistema operativo su cui viene installato.

I vantaggi sono i seguenti:
- L’installazione del VMM è più semplice (dato che è come un'applicazione).
- Si può fare riferimento al sistema operativo sottostante per la gestione delle periferiche e si possono utilizzare altri servizi del SO, come lo scheduling e la gestione dei dispositivi.
Uno svantaggio risulta nelle performance, peggiori rispetto al VMM di sistema. Un esempio di prodotti con VMM ospitato sono VirtualBox, User Mode Linux e VMware Fusion/player.

### 5. Problemi nella Realizzazione del VMM e Come Risolverli

I principali problemi che si possono presentare, nella realizzazione del VMM, sono dovuti ai **ring di protezione**. I ring di protezione, o modalità di esecuzione, sono uno strumento utilizzato dai processori per incrementare il livello di protezione tra i diversi componenti e separare i compiti. Soltiamente vi sono almeno 2 ring: il <ins>ring 0</ins> (supervisor o kernel) è l'unico in cui è possibile eseguire le istruzioni privilegiate della CPU; i <ins>ring >0</ins> (utente) sono quelli in cui non è possibile eseguire le istruzioni privilegiate. Poiché in un sistema virtualizzato con VMM di sistema, <ins>il VMM dev'essere l'unico componente in grado di mantenere in qualunque momento il pieno controllo dell'HW</ins>, esso è anche l'unico componente che può e deve eseguire a ring 0. Di conseguenza:
1. Si può avere **ring deprivileging**, quando il SO di una macchina virtuale si trova ad eseguire ad un ring inferiore (che non gli è proprio) e di conseguenza non può utilizzare le istruzioni privilegiate del processore.<br/>
Una possibile <ins>soluzione</ins> a questo problema è l'utilizzo del meccanismo **trap&emulate**, secondo cui quando un SO guest tenta di eseguire un'istruzione privilegiata, la CPU scatena una notifica (*trap*) al VMM e gli trasferisce il controllo. Dopodiché, il VMM controlla la correttezza della richiesta e ne emula (*emulate*) il comportamento.<br/>
Esempio: se le VM potessero eseguire le istruzioni privilegiate, un SO guest potrebbe chiamare la ```popf```, un'istruzione privilegiata che permette di disabilitare le interruzioni. Ma in questo modo verrebbero disabilitate le interruzioni di tutto il sistema, ed il VMM non potrebbe più riacquisire il controllo dell'HW. Invece, il comportamento desiderato è che venissero disabilitate solo le istruzioni della singola VM che ha effettuato tale chiamata, comportamento realizzabile tramite l'approccio *trap&emulate*.
2. Si può avere **ring compression**, se ad esempio il processore prevede due soli ring di protezione 0 ed 1. In questo caso, il VMM si troverà a ring 0, mentre sia SO guest che applicazioni si troveranno ad eseguire nello stesso ring utente 1, con *scarso livello di protezione* tra SO e applicazioni.
3. Si possono verificare problemi dovuti al **ring aliasing**, quando alcune istruzioni non privilegiate permettono di accedere in lettura ad alcuni registri la cui gestione dovrebbe essere riservata al VMM, con *possibili inconsistenze*. Ad esempio il registro CS contiene il current privilege level (CPL) e un SO potrebbe leggere un valore diverso rispetto a quello in cui pensa di eseguire.

### 6. Supporto HW alla Virtualizzazione

L'architettura di una CPU si dice **naturalmente virtualizzabile** se <ins>prevede l'invio di trap allo stato supervisor</ins> (ring 0) ogni volta che un livello di protezione differente tenta di eseguire istruzioni privilegiate. In questo caso la realizzazione del VMM è semplificata, in quanto l'approccio trap&emulate ha il support dell'HW, e vi è supporto all'esecuzione diretta (le istruzioni non privilegiate vengono eseguite direttamente dalle VM guest).

### 7. Realizzazione VMM in Architetture Non Virtualizzabili: FTB e Paravirtualizzazione, PRO e CONTRO

L’architettura della CPU si dice naturalmente virtualizzabile se prevede l’invio di trap allo stato supervisore per ogni istruzione privilegiata invocata da un livello di protezione diverso dal supervisore.

Se l’architettura della CPU è naturalmente virtualizzabile la realizzazione del VMM è semplificata: per ogni trap generato dal tentativo di esecuzione di istruzione privilegiata dal guest viene eseguita una routine di emulazione (seguendo l’approccio trap&emulate). 

È inoltre presente il supporto nativo all’esecuzione diretta (ad esempio Intel VT, AMD-V).
Non tutte le architetture sono naturalmente virtualizzabili: alcune istruzioni privilegiate di questa architettura invocate a livello user non provocano una trap, ma:

- vengono ignorate non consentendo l’intervento trasparente del VMM,
- in alcuni casi provocano il crash del sistema.

Se il processore non fornisce il supporto alla virtualizzazione, si utilizzano delle soluzioni software apposite, come fast binary translation e paravirtualizzazione.

1. **Fast Binary Translation (FTB)**, si basa sulla compilazione dinamica. Il VMM legge dinamicamente (a runtime) blocchi di istruzioni chiamati dalle VM guest, e <ins>sostituisce le chiamate ad istruzioni privilegiate con chiamate al VMM</ins>, ottenendo lo stesso significato semantico. Come per la compilazione dinamica, vi è la possibilità di salvare in cache i blocchi tradotti, per riutilizzi futuri.
- *Vantaggi*: ogni VM guest usa la stessa interfaccia fornita dall'architettura fisica, dunque è una copia esatta della macchina reale (virtualizzazione pura: non è necessario il porting del Sistema Operativo).
- *Svantaggi*: la traduzione dinamica è costosa, dunque le prestazioni ne risentono, e la struttura del VMM è più complessa, in quanto deve realizzare anche il layer relativo alla traduzione binaria.

2. **Paravirtualizzazione**, il VMM offre alle VM guest un'interfaccia differente (<ins>hypercall API</ins>) rispetto a quella fornita dalla macchina fisica, per l'accesso alle risorse. I SO guest quando vogliono eseguire istruzioni privilegiate, eseguono direttamente le hypercall, senza generare interruzioni.I kernel dei sistemi operativi guest devono essere modificati in modo da avere accesso all’intefaccia del
particolare VMM. Si ha una struttura del VMM semplificata, poiché non si deve occupare della traduzione dinamica dei tentativi di operazioni privilegiate dei guest.
Un esempio di architettura che utilizza la paravirtualizzazione è Xen.
- *Vantaggi*: prestazioni migliori rispetto a fast binary translation e VMM semplificato.
- *Svantaggi*: necessità di effettuare il porting dei SO guest (operazione preclusa a sistemi operativi proprietari, ad esempio famiglia Windows).

### 8. Illustrare i meccanismi di protezione introdotti nell’architettura x86

La protezione viene introdotta a partire dalla seconda generazione dell’architettura x86: viene effettuata una distinzione tra sistema operativo (che possiede controllo assoluto sulla macchina fisica sottostante) e le applicazioni (che interagiscono con le risorse fisiche effettuando una richiesta al sistema operativo e implementando il concetto di ring di protezione.

Viene utilizzato il registro **CS**, i cui due bit meno significativi vengono riservati per rappresentare il livello corrente di privilegio (**CPL**, Current Privilege Level). Sono possibili 4 ring, di cui:

- **Ring 0**: possiede i maggiori privilegi ed è destinato al kernel del sistema operativo.
- **Ring 3**: possiede i minori privilegi ed è quindi destinato alle applicazioni utente.

Normalmente si utilizzano comunemente soltanti il ring 0 e il ring 3, mentre gli altri due sono utilizzati in rari casi (e.g. IBM OS2), per mantenere la massima portabilità dei sistemi operativi verso processori con solo 2 ring di protezione. Per garantire protezione della CPU non è permesso a ring diversi dallo 0 di eseguire le istruzioni privilegiate e normalmente destinate solo al kernel del sistema operativo, in quanto sono critiche e potenzialmente pericolose.

Una qualsiasi violazione di questo comportamento può provocare un’eccezione, con immediato passaggio al sistema operativo, in grado di catturarla e gestirla opportunamente e terminando ad esempio l’applicazione in esecuzione.

Per garantire protezione della memoria si guarda il descrittore di ciascun segmento, presente in una tabella GDT o LDT: in particolare, nel descrittore sono indicati il livello di protezione richiesto **PL** e i vari permessi di accesso (r, w, x).

Se il valore di CPL è maggiore del valore del PL del segmento di codice che contiene l’istruzione invocata,
allora si ha una violazione dei vincoli di protezione, che provoca un’eccezione.

Per risolvere il problema del ring deprivileging viene dedicato il ring 0 al VMM e conseguentemente i sistemi operativi guest vengono collocati in ring a privilegi ridotti. Vengono comunemente utilizzate due tecniche:

- `0/1/3`: Consiste nello spostare il sistema operativo dal ring 0, dove nativamente dovrebbe trovarsi, al ring
1 a privilegio ridotto, lasciando le applicazioni nel ring 3 e installando il VMM sul ring 0. Questa tecnica
non è però compatibile con sistemi operativi a 64 bit. 

- `0/3/3`: Consiste nello spostare il sistema operativo
direttamente al ring applicativo, e cioè il 3, insieme alle applicazioni stesse, installando sul ring 0, come nella
tecnica precedente, il VMM (ring compression)

### 9. Migrazione di VM

Specialmente nei datacenter che forniscono server virtualizzati, la migrazione è utile per una gestione agile delle VM. In particolare è utile per far fronte a: variazioni dinamiche del carico (dunque è possibile effettuare load balancing), manutenzione online dei nodi, gestione finalizzata al risparmio energetico, disaster recovery.
La migrazione Live permette di spostare una VM da un nodo fisico ad un altro senza doverla spegnere, permettendo di mantenere attivi i servizi da essa forniti. Solitamente si cerca di minimizzare downtime (tempo in cui la macchina non risponde alle richieste degli utenti), tempo totale di migrazione e consumo di banda.
Migrazione Live tramite Precopy: ha l'obbiettivo di minimizzare il downtime.
1. *Individuazione* dei nodi coinvolti nella migrazione (sorgente e destinazione);
2. Allocazione ed *inizializzazione* di una VM container sul nodo di destinazione;
3. **Pre-copia iterativa** delle pagine:
- Prima si copiano tutte le pagine allocate in memoria sull'host sorgente, nell'host destinatario;
- Poi si effettua una copia iterativa delle dirty pages (ovvero le pagine modificate rispetto al ciclo precedente), finché non si raggiunge una soglia minima di pagine.
4. *Sospensione* della macchina sull'host sorgente e copia delle ultime dirty pages;
5. *Commit*: eliminazione della macchina dall'host sorgente;
6. *Resume* della macchina sull'host destinatario.
Alternative a pre-copy: post-copy (riduce tempo totale di migrazione e consumo di banda, ma ha downtime piuttosto elevato).

### 10. XEN: Architettura, Virtualizzazione Memoria (Paginazione, Memory Split, Balloon Process)

XEN è un VMM di sistema open source basato sulla paravirtualizzazione.

L'architettura di XEN prevede che il VMM (di sistema) esegua direttamente sull'HW. Le VM guest sono organizzate in domain: vi è il *domain 0* che è assegnato ad una VM speciale (separato dal VMM stesso), e vi sono i *domain utente* (>0) che sono le VM installate. Il VMM si occupa di virtualizzare CPU, memoria e dispositivi di I/O per ogni VM e fornisce un'interfaccia di controllo per la gestione delle risorse e l'amministrazione dei vari domain. Sfrutta la *paravirtualizzazione*: le VM guest eseguono direttamente le istruzioni non privilegiate ed effettuano hypercall per le istruzioni privilegiate.

##### Virtualizzazione della memoria
**Paginazione**: I SO guest gestiscono la *memoria virtuale* mediante paginazione (<ins>meccanismi gestiti da XEN -ring 0-, politiche da VM guest</ins>). Le tabelle delle pagine sono mappate nella memoria fisica da XEN, il quale è l'unico a potervi accedere in *scrittura*, mentre i SO guest possono accedervi in lettura. Per gli *update*, le VM effettuano delle richieste ed il VMM le controlla e le esegue.<br/>
**Memory Split**: XEN adotta il modello 0/1/3, con VMM a ring 0, SO guest a ring 1 e applicazioni a ring 3. Per aggiungere un ulteriore livello di protezione, viene adottato un meccanismo chiamato *memory split* secondo cui lo <ins>spazio di indirizzamento virtuale per ogni VM è strutturato in modo da contenere il codice di XEN (nei primi 64 MiB, ring 0), il codice del kernel (ring 1) e lo spazio utente (ring 3), in segmenti separati</ins>. Al momento della creazione di un processo, il SO ospitato richiede una tabella delle pagine a XEN, che ne restituisce una a cui sono state aggiunte le pagine del segmento di XEN, registrandola e acquisendovi il diritto di scrittura esclusivo. In questo modo, quando un SO guest tenta di aggiornarla, scatenerà una *protection fault*, che verrà catturato e gestito da XEN.<br/>
**Balloon Process**: Poiché la paginazione è completamente a carico delle VM guest, serve un meccanismo che <ins>consente al VMM di reclamare pagine di memoria meno utilizzate dalle altre VM</ins>. Per questo motivo, su ogni VM guest è sempre in esecuzione un balloon process che comunica col VMM e, in caso di necessità, può essere chiamato per *gonfiarsi* e richiedere al proprio SO delle pagine, fornendole successivamente al VMM.

### 11. XEN: Virtualizzazione CPU, Virtualizzazione Driver, Virtualizzazione delle Interruzioni, Migrazione

**Virtualizzazione CPU**: Il VMM definisce un'architettura virtuale simile a quella del processore, in cui però le istruzioni privilegiate sono sostituite con hypercall (necessità di porting dei SO guest): l'invocazione di una hypercall determina il passaggio da ring 1 a ring 0. Due clock: real-time (processore), virtual-time (VM).

**Virtualizzazione Driver**: Per consentire alle VM guest di accedere ai dispositivi disponibili a livello HW, XEN virtualizza l'interfaccia di ciascuno, tramite 2 tipi di driver:
- **back-end driver**, è il driver vero e proprio, solitamente installato nel domain 0;
- **front-end driver**, è il driver "astratto", semplificato e generico installato nel kernel del SO guest, che all'occorrenza si collega al back-end specifico.

Per la gestione delle richieste viene utilizzata una struttura ad anello chiamata "asyncronous I/O ring" (buffer FIFO circolare), in cui i front-end driver depositano le richieste, che vengono estratte dal back-end driver. Questa soluzione garantisce portabilità, isolamento e semplificazione del VMM.

**Virtualizzazione delle Interruzioni**: Ogni interruzione viene gestita direttamente dal SO guest, ad eccezione dei page fault, in quanto questa richiede l'accesso al registro CR2 (accessibile solo a ring 0), che contiene l'indirizzo di chi l'ha provocato. Dunque la routine di gestione dei page fault prevede che il VMM legga il valore di CR2, lo copi in una variabile del SO guest, e vi restituiscac il controllo.

**Migrazione Live**: la migrazione live su XEN è guest based, e avviene sfruttando un demone che si trova nel domain 0 (del server sorgente). Si adotta la pre-copy con compressione delle pagine per ridurre l'occupazione di banda.

<h1 align="center">PROTEZIONE</h1>

### 1. Cosa si Intende per Sistema di Protezione. Modello, Politiche, Meccanismi

Un sistema di protezione permette di definire delle tecniche di controllo degli accessi. Questo si esprime tramite 3 concetti fondamentali:
##### Modello
Definisce:
- **oggetti**, ovvero la <ins>parte passiva</ins>, le risorse fisiche e logiche, ad esempio i file;
- **soggetti**, ovvero la <ins>parte attiva</ins>, le entità che possono richiedere l'accesso alle risorse, ad esempio utenti e processi;
- **diritti di accesso**, ovvero le <ins>operazioni</ins> con cui i soggetti possono operare sugli oggetti, ad esempio lettura e scrittura.
NB: un soggetto può avere diritti di accesso sia per oggetti che per soggetti.
##### Politiche
Definiscono le *regole* con cui i soggetti possono accedere agli oggetti. Si classificano in 3 tipologie:
- **Discretionary Access Control (DAC)**, prevede che il <ins>proprietario</ins> di un oggetto ne controlli i diritti di accesso (gestione delle politiche decentralizzata, come accade in UNIX);
- **Mandatory Access Control (MAC)**, prevede che i diritti vengano decisi in modo <ins>centralizzato</ins> (tipico dei sistemi ad alta sicurezza, ad esempio enti governativi);
- **Role Based Access Control (RBAC)**, prevede che i diritti di accesso alle risorse vengano assegnati in base al <ins>ruolo, che viene assegnato in modo centralizzato</ins> (gli utenti possono appartenere a diversi ruoli).
##### Meccanismi
Sono gli strumenti messi a disposizione dal sistema di protezione per imporre una determinata politica e vanno realizzati per rispettare:
- **flessibilità** del sistema di protezione, ovvero devono essere abbastanza generali da permettere l'applicazione di diverse politiche;
- **separazione tra meccanismi e politiche**, secondo cui la politica definisce "cosa va fatto" ed i meccanismi "come va fatto".

###### Esempio UNIX
Politica DAC: l'utente definisce la *politica*, ovvero il valore dei bit di protezione per ogni oggetto di sua proprietà, ed il SO fornisce un *meccanismo* per definire ed interpretare per ciascun oggetto i bit di protezione.

### 2. Cos'è il Principio del Minimo Privilegio e Come si Può Implementare

Se il Principio del Minimo Privilegio (Principle Of Least Authority - POLA) viene rispettato, ad <ins>ogni soggetto</ins> sono garantiti i <ins>diritti di accesso dei soli oggetti strettamente necessari alla sua esecuzione<ins>. Il rispetto di questo principio è desiderabile a prescindere dalla politica adottata.

Per implementarlo è possibile adottare un'*associazione processo-dominio dinamica*, che permetta di effettuare, a tempo di esecuzione del processo, il passaggio da un dominio ad un altro, in base alle risorse ad esso necessario in qualsiasi istante della sua esecuzione.

### 3. Cos'è il Dominio di Protezione. Dominio statico/dinamico e Cambio di Dominio

Un dominio di protezione definisce un insieme di coppie <oggetto, diritti di accesso>, che rappresenta l'<ins>ambiente di protezione nel quale un certo soggetto esegue</ins>. Il dominio di protezione è infatti univoco per ciascun soggetto, e in ogni istante della sua esecuzione, un soggetto (processo) è associato ad uno ed un solo dominio, e può accedere solo agli oggetti specificati nel suo dominio, con i relativi diritti.

L'associazione tra processo e dominio può essere:
- **statica**, se rimane fissa durante l'intera esecuzione del processo. 
- **dinamica**, se può variare nel corso dell'esecuzione del processo.
Poiché a tempo di esecuzione, l'insieme globale delle risorse che un processo potrà usare può non essere conosciuto a priori, e l'insieme minimo delle risorse a lui necessarie cambia dinamicamente durante l'esecuzione, l'associazione statica non permette di realizzare il Principio del Minimo Privilegio. Al contrario, ciò è possibile con l'associazione dinamica, tuttavia occorre un meccanismo di cambio di dominio.

Esempi: cambio di dominio relativo all'esecuzione di system call (2 ring, protezione tra kernel e utente, ma non tra diversi utenti); cambio di dominio in UNIX, realizzato tramite il bit set-uid che, se abilitato, permette al processo che esegue il file di passare nel dominio del proprietario del file.

### 4. Matrice degli Accessi e Com'è Possibile Rappresentarla Concretamente (PRO e CONTRO, e Possibile Soluzione)

La matrice degli accessi permette di rappresentare, a livello astratto, lo <ins>stato di protezione</ins> di un sistema in un determinato istante: ad esempio si possono utilizzare le righe per indicare i soggetti e le colonne per gli oggetti, mentre i singoli elementi contengono i vari diritti di accesso. Offre ai meccanismi le informazioni che gli consentono di verificare il rispetto dei vincoli di accesso.

Solitamente il numero dei soggetti e soprattutto degli oggetti tende ad essere molto grande (e i diritti di accesso generalmente sono sparsi). Dunque, la matrice degli accessi non può essere realizzata come un'unica struttura Ns x No, in quanto ciò non sarebbe ottimale per l'occupazione della memoria né per l'efficienza negli accessi. Per questo motivo, la realizzazione concreta dev'essere ottimizzata, ed esistono 2 approcci:
- **Access Control List (ACL)**, si basa su una rappresentazione per <ins>colonne</ins> e prevede che ad ogni <ins>oggetto</ins> sia associata una lista di coppie <soggetto, insieme-dei-diritti> (o <soggetto, gruppo, insieme-dei-diritti>), solo per i soggetti con un insieme non vuoto di diritti per l'oggetto.
Quando dev'essere fatta un'operazione M su un oggetto Oj dal soggetto Si, il meccanismo di protezione cerca nell'ACL corrispondente all'oggetto Oj l'entry corrispondente al soggetto Si e controlla se è presente il diritto di eseguire M. La ricerca può essere fatta anche su una lista che contiene i diritti di accesso comuni a tutti i soggetti.
La <ins>revoca</ins> di un diritto di accesso è molto <ins>semplice</ins> con ACL, in quanto basta fare riferimento all'oggetto coinvolto.
- **Capability List (CL)**, si basa su una rappresentazione per <ins>righe</ins> e prevede che per ogni <ins>soggetto</ins> si abbia una lista di coppie <oggetto, insieme-dei-diritti>, che prendono nome di *capability* (capability = coppia).
Per proteggere le CL da manomissioni, esse vengono memorizzate nello spazio del kernel e l'utente può far riferimento solo ad un puntatore che identifica la sua posizione nella lista.
La <ins>revoca</ins> di un diritto di accesso è più <ins>complessa</ins> perché è necessario verificare, per ogni dominio (soggetto), se contiene capability che fanno riferimento all'oggetto considerato.

Entrambe le soluzioni presentano **problemi di efficienza**: con le ACL i diritti di accesso di un particolare soggetto sono sparsi nelle varie ACL; con CL, l'informazione relativa a tutti i diritti di accesso applicabili ad un certo oggetto è sparsa nelle varie CL.

**Soluzione Ibrida**: vengono combinati i due metodi. La ACL viene memorizzata in <ins>memoria persistente</ins> (secondaria) e, quando un soggetto tenta di accedere ad un oggetto per la prima volta, se il diritto invocato è presente nella ACL, viene restituita la CL relativa al soggetto richiedente, e salvata in <ins>memoria volatile</ins> (RAM). In questo modo il soggetto può accedere all'oggetto più volte senza dover analizzare nuovamente la ACL. Dopo l'ultimo accesso, la CL viene distrutta dalla memoria volatile.

### 5. Diritti di Accesso: Copy Flag (*), Owner, Control, Switch. È Possibile Capire Quale Politica si sta Utilizzando?

**Copy Flag** (\*): è un diritto di accesso esercitato da un soggetto su un particolare diritto di accesso per un oggetto, che permette la propagazione di tale diritto ad altri soggetti. La propagazione può essere realizzata in due modi: *trasferimento* (il soggetto iniziale perde il diritto), e *copia* (il soggetto iniziale mantiene il diritto).

**Owner**: possedere tale diritto su un oggetto permette di assegnare e revocare un qualunque diritto di accesso su tale oggetto ad altri soggetti.

**Control**: se un soggetto S1 possiede tale diritto su un altro soggetto S2, S1 può revocare a S2 un qualunque diritto di accesso per oggetti nel suo dominio (di S2).

**Switch**: se un soggetto possiede tale diritto su un altro soggetto, può spostarsi nel dominio di quest'ultimo.

In certi casi è possibile capire quale politica il sistema di protezione sta adottando in base alla presenza di certi diritti. Copy flag e owner, ad esempio, indicano esplicitamente l'utilizzo di una politica DAC (decentralizzata).

### 6. Sicurezza Multilivello: Modello Bell-La Padula e BIBA + Esempio Cavallo di Troia

In alcuni ambienti è necessario un controllo più stretto sulle regole di accesso alle risorse (es: militare). I sistemi di sicurezza multilivello prevedono che vengano stabilite regole generali non modificabili senza aver ottenuto dei permessi speciali (basato su politica MAC, ovvero controllo degli accessi obbligatorio). In un sistema di sicurezza multilivello, i soggetti e gli oggetti sono classificati in livelli (classi di accesso) e vengono imposte delle regole di sicurezza che controllano il flusso delle informazioni tra i livelli.

##### Modello Bell-La Padula
È progettato per garantire la segretezza (confidenzialità) dei dati, ma non l'integrità. Associa al sistema di protezione (matrice degli accessi), un modello di sicurezza multilivello, che prevede 2 regole:
1. **Semplice sicurezza**, permette ad un processo in esecuzione ad un determinato livello, di <ins>leggere solo oggetti di livello pari o inferiore</ins>;
2. **Star (o di integrità)**, permette ad un processo in esecuzione ad un determinato livello, di <ins>scrivere solo oggetti di livello pari o superiore</ins>.

Esempio di **difesa da un Trojan**, con modello Bell-La Padula:
*S1* possiede un file *F1* da proteggere, con permessi di lettura/scrittura che appartengono solo a lui (*S1*);
*S2* è ostile e vuole rubarli, e possiede un file eseguibile *CT* (Cavallo di Troia), che ha installato nel sistema, assieme ad un file *F2* che usa come "tasca posteriore".
Stato della ACL:
- *S2* ha permessi di lettura/scrittura per *F2* (tasca posteriore);
- *S2* dà a *S1* il permesso di scrittura su *F2*;
- *S2* dà a *S1* il permesso di esecuzione su *CT*;
- *F1* (file da proteggere) è leggibile solo da *S1*.

*S2* induce *S1* ad eseguire *CT* che, essendo eseguito a nome di *S1*, può leggere *F1* e scrivere su *F2*. In quanto sia lettura che scrittura soddisfano i vincoli della ACL.<br/>
Tuttavia, se il sistema prevedesse il modello di sicurezza multilivello Bell-La Padula, e ci fossero ad esempio 2 livelli (*riservato*, per processi e file di *S1*, e *pubblico*, per processi e file di *S2*): il processo che esegue *CT* assumerebbe il livello di *S1* (riservato), dunque potrebbe leggere il file *F1* da proteggere, in quanto di pari livello (proprietà di semplice sicurezza rispettata); ma non potrebbe scrivere sul file *F2*, in quanto di livello inferiore (proprietà star violata). Dunque l'accesso, nonostante è consentito dalla ACL, viene negato.

##### Modello BIBA
È progettato per garantire l'integrità dei dati, ma non la segretezza. Prevede anch'esso 2 regole:
1. **Semplice sicurezza**, permette ad un processo in esecuzione ad un determinato livello, di <ins>scrivere solo oggetti di livello pari o inferiore</ins>;
2. **Star (o di integrità)**, permette ad un processo in esecuzione ad un determinato livello, di <ins>leggere solo oggetti di livello pari o superiore</ins>.

I modelli Bell-La Padula e BIBA sono in conflitto e non possono essere utilizzati contemporaneamente. Le politiche di sicurezza multilivello coesistono con le regole imposte dal sistema di protezione (ACL/CL) e hanno la *priorità* su quest'ultime.

### 7. Cosa sono i Sistemi Trusted, Quali Sono i Componenti Principali e Che Proprietà Devono Avere

Un sistema trusted è un sistema per il quale è possibile definire formalmente dei requisiti di sicurezza. L'architettura di tale sistema prevede 2 componenti fondamentali:
- **Reference Monitor (RM)**, è un elemento di controllo realizzato dall'HW e dal SO, che <ins>regola l'accesso</ins> dei soggetti agli oggetti <ins>in base alle regole di sicurezza</ins> (ad esempio fornite da un modello di sicurezza multilivello, tipo Bell-La Padula).
- **Trusted Computing Base (TCB)**, è un elemento che <ins>contiene i livelli di sicurezza</ins> di soggetti (privilegi di sicurezza) e oggetti (classificazione rispetto alla sicurezza).

I Sistemi Trusted devono rispettare le seguenti proprietà:
- **mediazione completa**, ovvero le regole di sicurezza devono essere applicate ad ogni accesso alle risorse, e non solo. Dunque, essendo questa un'operazione piuttosto frequente, per motivi di efficienza è necessario che la soluzione venga implementata (almeno parzialmente) via HW;
- **isolamento**, ovvero sia RM che TCB devono essere isolati e protetti rispetto a modifiche non autorizzate (anche ad esempio da parte del kernel del SO);
- **verificabilità**, ovvero dev'essere possibile dimostrare formalmente che il RM esegua correttamente il suo compito (imponendo il rispetto delle regole di sicurezza, e fornendo mediazione completa ed isolamento). Questo solitamente è un requisito difficile da soddisfare in un sistema general-purpose.

**Audit File**: è una specie di <ins>file di log</ins>, che mantiene tutte le informazioni sulle operazioni eseguite più importanti e di interesse dal punto di vista della sicurezza del sistema, ad esempio modifiche autorizzate alla TCB o tentativi di violazione.

###### Classificazione della Sicurezza dei Sistemi di Calcolo
Secondo l'Orange Book (documento pubblicato dal Dipartimento della Difesa americano), la sicurezza di un sistema viene classificata in base a 4 categorie:
- Categoria **D: Minimal Protection**. Non prevede sicurezza. Es: MS-DOS.
- Categoria **C: Discretionary Protection**. Es: Unix.
- Categoria **B: Mandatory Protection**. Introduzione di livelli di sicurezza (es: Bell-La Padula).
- Categoria **A: Verified Protection**.

<h1 align="center">PROGRAMMAZIONE CONCORRENTE</h1>

### 1. Descrivere la Tassonomia di Flynn

La tassonomia di Flynn è la più usata classificazione dei sistemi di calcolo e si basa su 2 concetti: parallelismo a livello di istruzioni (Single Instruction stream, o Multiple Instruction stream) e parallelismo a livello di dati (Single Data stream o Multiple Data stream):

- **Single Instruction, Single Data (SISD)**, riguarda gli elaboratori monoprocessore (es: macchina di Von Neumann);
- **Single Instruction, Multiple Data (SIMD)**, prevede molte unità di elaborazione che eseguono la stessa istruzione su una moltitudine di dati differenti (es: elaboratori vettoriali: GPU);
- **Multiple Instruction, Single Data (MISD)**, il sistema è in grado di gestire un unico flusso di dati che ad ogni istante può essere elaborato da più istruzioni differenti (es: pipelined computer);
- **Multiple Instruction, Multiple Data (MIMD)**, insieme di nodi di elaborazione ognuno dei quali può eseguire flussi di istruzioni diverse, su dati diversi (es: multiprocessori).

### 2. Descrivere le Possibili Interazioni tra Processi

Esistono 3 possibili tipi di interazione fra processi:
1. **Cooperazione**, comprende tutte le interazioni <ins>prevedibili e desiderate</ins>, che sono in qualche modo dettate dall'algoritmo (ovvero date dagli archi del grafo di precedenza ad ordinamento parziale). Si può esprimere in 2 modi, entrambi dei quali esprimono un *vincolo di precedenza*:
- mediante <ins>segnali temporali</ins>, ovvero pura sincronizzazione;
- mediante <ins>scambio di dati</ins>, ovvero con comunicazione.
2. **Competizione**, consiste in un'interazione <ins>prevedibile ma non desiderata</ins>, in quanto non fa parte dell'algoritmo, ma è imposta dai limiti delle risorse a cui i processi devono accedere, ad esempio una risorsa che può essere acceduta solo in modo mutuamente esclusivo. In questo caso si prevede il concetto di *sezione critica*, ovvero la sequenza di istruzioni con cui un processo accede ad un oggetto condiviso mutuamente esclusivo. Ad una risorsa possono essere associate anche più di una sezione critica, di classi differenti.
3. **Interferenza**, consiste in un'interazione <ins>non prevedibile e non desiderata</ins> solitamente causata da *errori del programmatore* (es: deadlock).

### 3. Costrutti Linguistici per la Specifica della Concorrenza

Il linguaggio concorrente deve fornire costrutti che consentano di gestire i processi. Esistono 2 modelli differenti:
- **Fork/Join**, comprende una primitiva <ins>fork</ins> per la *creazione* e l'*attivazione* di un processo che eseguirà in parallelo, ed una primitiva <ins>join</ins> per la sincronizzazione con la terminazione di un processo. Nel grafo di precedenza, una fork coincide con una biforcazione, mentre una join con un nodo avente due precedenti.
- **Cobegin/Coend**, comprende una primitiva <ins>cobegin</ins> per la specifica di un *blocco di codice che deve essere eseguito in parallelo*, ed una primitiva <ins>coend</ins> per la specifica della terminazione del blocco. Le istruzioni contenute all'interno vengono eseguite in parallelo ed è possibile innestare dei blocchi uno dentro l'altro.

### 4. Proprietà dei Programmi Sequenziali e Concorrenti

Una delle attività più importanti per chi sviluppa programmi è la verifica di correttezza dei programmi realizzati.

Proprietà dei Programmi *Sequenziali*:
1. **Safety**, ovvero la correttezza del risultato finale (il programma non entrerà mai in uno stato in cui le variabili assumono valori non desiderati).
2. **Liveness**, ovvero la terminazione del programma (prima o poi il programma entrerà in uno stato in cui le variabili assumono valori desiderati).

Proprietà dei Programmi *Non Sequenziali*:
1. **Safety**, correttezza del risultato finale.
2. **Liveness**, terminazione del programma.
3. **Mutua Esclusione nell'Accesso a Risorse Condivise**, ovvero per ogni esecuzione non si deve mai verificare che più di un processo acceda contemporaneamente ad una stessa risorsa (mutuamente esclusiva).
4. **Assenza di Deadlock**.
5. **Assenza di Starvation**, ovvero ciascun processo che richiede l'accesso ad una certa risorsa, prima o poi lo otterrà.

<h1 align="center">MODELLO A MEMORIA COMUNE</h1>

### 1. Aspetti Caratterizzanti del Modello a Memoria Comune e Regione Critica Condizionale

Nel modello a memoria comune ogni interazione tra i processi avviene tramite oggetti contenuti in memoria comune. Ogni applicazione è vista come un insieme di componenti *attivi* (processi) e componenti *passivi* (risorse). I processi possono avere diritto di accesso sulle risorse, di cui necessitano per portare a termine il loro compito. 

È presente un **gestore delle risorse** che definisce, per ogni istante t, l'insieme SR(t) dei processi che, all'instante t, hanno diritto di operare su R e i cui compiti sono:
- mantenere aggiornato l'insieme `SR(t)`, cioè lo stato di allocazione della risorsa;
- fornire i meccanismi che il processo può utilizzare per acquisire il permesso di operare sulla risorsa e di far parte dell'insieme `SR(t)`, per rilasciare tale diritto quando non è più necessario;
- implementare la strategia di allocazione di risorsa, ovvero stabilire quale processo, quando e per quanto tempo può utilizzare tale risorsa. 

Una risorsa può essere:
- **dedicata**: se la cardinalità `SR(t) ≤ 1` (uno o zero processi utilizzano quella risorsa all'istante `t`)
- **condivisa**: in caso contrario
- **allocata staticamente**: se `SR(t)` è una costante, quindi l'insieme dei processi che possono utilizzare tale risorsa rimane lo stesso per qualsiasi istante `t`.
- **allocata dinamicamente**: se `SR(t)` è in funzione del tempo. È necessario prevedere un gestore che implementa le funzioni di _richiesta_ e _rilascio_ di una determinata risorsa.

Il modello a memoria comune prevede che `R` sia allocata come **risorsa condivisa**, e deve garantire che l'accesso alla risorsa `R` avvenga in modo _non divisibile_. A tal scopo le funzioni di accesso alla risorsa devono essere programmate come una _classe di sezioni critiche_, utilizzando i meccanismi di sincronizzazione offerti dal linguaggio di programmazione e supportati dalla macchina concorrente.

**Regione Critica Condizionale**: formalismo che consente di *esprimere* qualunque vincolo di sincronizzazione. Si esprime come: 

```
region R << Sa; when(C) Sb; >>
``` 
dove `R` è la risorsa condivisa, `Sa` ed `Sb` sono istruzioni, e `C` una condizione da verificare.<br/>
Il corpo (tra virgolette) consiste in un'operazione sulla risorsa `R` e rappresenta una sezione critica che deve essere eseguita in mutua esclusione con le altre operazioni definite su `R`. Una volta terminata `Sa` viene valutata la condizione `C`:
- se è *vera* si prosegue con `Sb`;
- se è *falsa* si <ins>attende</ins> che `C` diventi vera. Quando `C` diventa vera, allora si prosegue con `Sb`.

### 2. Semaforo: Definizione e Proprietà

Il semaforo è uno strumento linguistico di basso livello che consente di *risolvere* qualunque problema di sincronizzazione nel modello a memoria comune.
Il suo meccanismo è realizzato dal kernel della macchina concorrente, e l'attesa può essere implementata mediante i meccanismi di gestione dei thread offerti dal kernel. Viene utilizzato per implementare strumenti di sincronizzazione di più alto livello (ad esempio le condition).

**Definizione Semaforo**: il semaforo ```S``` è una variabile intera non negativa ```val ≥ 0```, alla quale è possibile accedere solo mediante due operazioni mutuamente esclusive ```P``` e ```V```:
- ```void P(sem S): region S << when(C > 0) S.val-- >>```
- ```void V(sem S): region S << S.val++ >>```

Il semaforo viene associato ad una risorsa e, quando un processo vuole operare su tale risorsa, esso chiama una P (down/richiesta):
- se il valore del semaforo è positivo, il processo lo decrementa, esegue le sue operazioni, dopodiché chiama una V (up/rilascio);
- altrimenti (se il valore del semaforo è 0), si mette in attesa finché un altro processo, che sta attualmente usando la risorsa gestita dal semaforo, non chiama una V, incrementandone il valore.

**Proprietà del Semaforo**: dato un semaforo ```S```, siano ```val``` il suo valore (intero non negativo), ```I``` il valore `≥0` a cui viene inizializzato, ```nv``` il numero di volte che l'operazione V(S) è stata eseguita, ```np``` il numero di volte che P(S) è stata eseguita.

**Relazione di Invarianza**: ad ogni istante è possibile esprimere il valore del semaforo come 

```val = I + nv - np``` 

da cui (poiché val `≥0`) 

```I + nv - np ≥ 0``` 

dunque: 

```I + nv ≥ np``` (Relazione di Invarianza).<br/>

La relazione di invarianza è <ins>sempre soddisfatta</ins> per ogni semaforo.

### 3. Semaforo di Mutua Esclusione + Dimostrazione

Il Semaforo di Mutua Esclusione (o semaforo binario), viene <ins>inizializzato a 1</ins> e viene utilizzato per realizzare le sezioni critiche di una stessa classe, seguendo il <ins>protocollo: prima viene eseguita una P, poi una V</ins>, ovvero ```P(mutex); <sezione_critica>; V(mutex);```, dove mutex è un semaforo inizializzato a 1.

**Ipotesi**: Il semaforo è inizializzato a 1, e vengono eseguite prima la P poi la V.<br/>
**Tesi**:
1. le sezioni critiche della stessa classe vengono eseguite in mutua esclusione;
2. non devono verificarsi deadlock;
3. un processo che non sta eseguendo una sezione critica non deve impedire agli altri di eseguire la stessa sezione critica (o sezioni della stessa classe).

#### Dimostrazione di 1
La tesi di mutua esclusione equivale a dire che il <ins>numero di processi nella sezione critica</ins> Nsez è maggiore o uguale a 0, e minore o uguale a 1, ovvero ```Nsez ≥ 0 e 1 ≥ Nsez```.

Dato che è necessaria una P per entrare nella sezione critica, ed una V per uscire, si ha che il numero dei processi nella sezione critica è dato dal numero di volte in cui è stata eseguita una P, meno il numero di volte in cui è stata eseguita una v, ovvero: ```Nsez = np - nv```.<br/>
Ma dalla Relazione di Invarianza sappiamo che (I = 1): ```1 + nv ≥ np```, dunque ```1 ≥ np - nv```, ovvero ```1 ≥ Nsez```.<br/>
Inoltre, poiché il protocollo impone che P(mutex) preceda V(mutex), sappiamo che in qualunque istante dell'esecuzione ```np ≥ nv```, dunque ```np - nv ≥ 0```, ovvero ```Nsez ≥ 0```. □

#### Dimostrazione di 2
La tesi è l'assenza di deadlock, che dimostriamo per <ins>assurdo</ins>. Se ci fosse un deadlock:
1. tutti i processi sarebbero in attesa su P(mutex), portando il contatore del semaforo a 0, dunque ```val = 0```;
2. nessun processo sarebbe nella sezione critica, ovvero ```Nsez = np - nv = 0```.

Sapendo che `val = I + nv - np`, sostituendo otteniamo ```val = 1 - (np - nv)```, ovvero ```val = 1 - Nsez```, ma se `val = 0` e `Nsez = 0`, otteniamo ```0 = 1 - 0```, che è impossibile (assurdo). □

#### Dimostrazione di 3
La tesi prevede che non ci siano processi in sezione critica, ovvero ```Nsez = 0```.

Sostituendo nella relazione di invarianza otteniamo che: ```val = 1 - 0 = 1```, ovvero <ins>P non è bloccante</ins> (in quanto la P si blocca solo se `val = 0`). □

### 4. Semaforo Evento + Dimostrazione

Il semaforo evento è un semaforo binario utilizzato per imporre un <ins>vincolo di precedenza</ins> tra le operazioni dei processi.
Dato un processo *p* che esegue un'operazione *a*, si vuole che *a* possa essere eseguita solo dopo che un altro processo *q* abbia eseguito un'operazione *b*.
Il semaforo evento S è <ins>inizializzato a 0</ins> e segue il <ins>protocollo: prima di eseguire *a* il processo *p* esegue P(S); il processo *q* dopo aver eseguito *b* esegue V(S)</ins>.

**Ipotesi**: il semaforo è inizializzato a 0, e i 2 processi seguono il protocollo definito ```p: P(S); a;q: b; V(S);```.
**Tesi**: *a* viene eseguita sempre prima di *b*.

###### Dimostrazione
Dimostriamo la tesi per assurdo. Supponiamo che sia possibile che *a* venga eseguita in un istante precedente a quello in cui viene eseguita *b*. In questo modo avremmo che è stata eseguita una V(S) ma non una P(S), ovvero ```nv = 1``` e ```np = 0```.<br/>
Ma per la relazione di invarianza, sappiamo che `I + nv ≥ np`, ovvero ```0 + 0 ≥ 1```, che è impossibile (assurdo). □

### 5. Problema del Rendez-Vous + Barriera di Sincronizzazione

**Problema del Rendez-Vous**: si considerino due processi *A* e *B* che devono eseguire rispettivamente *a1*, *a2* e *b1*, *b2*, con il vincolo che l'esecuzione di *a2* e *b2* richieda che siano state completate sia *a1* che *b1*.

**Soluzione**: per risolvere questo problema si possono introdurre due semafori evento (ovvero inizializzati a val = 0) S1 e S2. Il processo *A* esegue in sequenza ```a1; V(S2); P(S1); a2;```, mentre il processo B esegue in sequenza ```b1; V(S1); P(S2); b2;```. In questo modo il processo che termina per primo si blocca sulla P in attesa dell'altro processo, rispettando i vincoli di precedenza.

**Generalizzazione del Problema del Rendez-Vous**: se i processi sono N > 2, è necessaria una struttura più complessa, chiamata *barriera di sincronizzazione*.

**Barriera di Sincronizzazione**: strumento che permette di subordinare l'esecuzione di una serie di operazioni *Pib* (i = 1, ..., N) al completamento di una serie di operazioni *Pia* (i = 1, ..., N).<br/>
La barriera è composta da:
- un semaforo binario <ins>mutex, inizializzato a 1</ins>;
- un semaforo evento <ins>barrier, inizializzato a 0</ins>;
- un <ins>contatore done, inizializzato a 0</ins>, che rappresenta il numero di processi che hanno completato la prima operazione (*Pia*).

Implementazione in pseudo-C del processo i-esimo:
```C
<operazione Pia>
P(mutex);
done++;
if(done == N)
	V(barrier);
V(mutex);
P(barrier);
V(barrier);
<operazione Pib>
```
In questo modo ogni processo attende la V(barrier) eseguita dall'ultimo processo (N-esimo) che completa la propria operazione *Pia*, prima di chiamare le rispettive V e iniziare la sequenza di risveglio degli N processi, facendo tornare il semaforo barrier a 0.

### 6. Descrivere l'Implementazione di un Semaforo nel Kernel di un Sistema Monoprocessore

Un semaforo può essere rappresentato come una struttura dati contenente un contatore *c* ed una coda *q* (politica FIFO). Una *P* su un semaforo con *c* == 0 sospende il processo corrente *p* e lo inserisce in *q* mediante una push; altrimenti, se *c* > 0, il contatore *c* viene decrementato. Una *V* su un semaforo con la coda *q* vuota incrementa il contatore, mentre se *q* non è vuota estra un processo *p* da *q* mediante una pop.

Implementazione in pseudo-C, supponendo che le <ins>interruzioni</ins> siano <ins>disabilitate</ins> durante l'esecuzione di *P* e *V*, in modo da garantire l'atomicità:
```C
typedef struct{
	int c;
	queue q;
} semaphore;

void P(semaphore s){
if (s.c > 0) {
s.c--;
} else {
// sospensione del processo corrente p, nella coda s.q
}
}
void V(semaphore s){
if (!isEmpty(s.q)){
// estrazione del primo processo p in attesa, dalla coda s.q
// risveglio del processo p
} else {
s.c++;
}
}
```
NB: l'implementazione di *P* e *V* è realizzata dal kernel della macchina concorrente e dipende dal tipo di architettura HW (monoprocessore, multiprocessore, ...) e da come il kernel rappresenta e gestisce i processi concorrenti.

<h1 align="center">NUCLEO DI UN SISTEMA MULTIPROGRAMMATO (MEMORIA COMUNE)</h1>

### 1. Spiegare Cos'è il Nucleo e quali sono le sue Funzioni Fondamentali

Il nucleo (o kernel) è il modulo realizzato in SW, HW o FW che supporta il concetto di processo e realizza gli strumenti necessari per la loro gestione e per la loro sincronizzazione. È il livello più interno di qualunque sistema basato su processi ed è l'unico conscio dell'esistenza delle interruzioni (sono invisibili ai processi).<br/>
Caratteristiche fondamentali del nucleo:
- **efficienza**, in quanto condiziona l'intera struttura a processi (alcuni sistemi prevedono l'esecuzione di operazioni del nucleo su hardware o tramite microprogrammi);
- **dimensioni ridotte**, in quanto le funzioni richieste al nucleo sono estremamente semplici;
- **separazione meccanismi e politiche**, il nucleo deve il più possibile contenere solo *meccanismi*, consentendo ai processi (in base ai meccanismi offerti dal nucleo) di scegliere ed applicare diverse politiche di gestione a seconda del tipo di applicazione.

Stati di un processo (in un sistema in cui il numero di processi è maggiore del numero delle unità di elaborazione):
- **esecuzione** (running), quando al processo è assegnata l'unità di elaborazione;
- **pronto** (ready), quando al processo è revocata l'unità di elaborazione;
- **bloccato** (waiting), quando il processo non è attivo (P sospensiva).
Quando un processo perde il controllo del processore, il suo <ins>contesto</ins> (ovvero il *contenuto dei registri del processore*) viene salvato nel <ins>descrittore</ins> (un'*area di memoria associata al processo*).

Le funzioni fondamentali del nucleo riguardano la gestione delle transizioni di stato dei processi, in particolare:
1. Gestire il <ins>salvataggio ed il ripristino dei contesti dei processi</ins>, ovvero trasferire le informazioni dai registri della CPU al descrittore, quando esso passa dallo stato di esecuzione allo stato di pronto o bloccato.
2. Effettuare lo <ins>scheduling della CPU</ins>, ovvero scegliere a quale processo assegnare l'unità di elaborazione.
3. Gestire le <ins>interruzioni dei dispositivi</ins> esterni, traducendo i processi coinvolti da bloccati a pronti.
4. Realizzare i <ins>meccanismi di sincronizzazione tra processi</ins>.

### 2. Spiegare Cos'è il Context Switch e Quali Funzioni deve Implementare il Nucleo per Realizzarlo

Il cambio di contesto (context switch) è un'operazione realizzata dal kernel del SO, che permette a più processi di condividere la CPU in un sistema multitasking. In particolare, il kernel può essere suddiviso in 2 livelli:
- *livello inferiore*, implementa i meccanismi per realizzare il cambio di contesto;
- *livello superiore*, fornisce le funzioni (risposta alle interruzioni, primitive per creazione, eliminazione e sincronizzazione dei processi) direttamente utilizzabili dai processi.

Il cambio di contesto permette di effettuare:
- **Salvataggio_stato**, prevede il salvataggio del contesto del processo in esecuzione (informazioni contenute nei registri del processore) nel suo descrittore (area di memoria), e l'inserimento del descrittore nella coda dei processi bloccati o dei processi pronti.
```C
void Salvataggio_stato() {
int j;
j = processo_in_esecuzione;
descrittori[j].contesto = <valori_registri_CPU>;
}
```
- **Assegnazione_CPU**, prevede la rimozione del processo a maggior priorità dalla coda dei processi pronti e il caricamento dell'identificatore di tale processo nel registro contenente il processo in esecuzione.
```C
void Assegnazione_CPU() {// scheduling: algoritmo con priorità
int k = 0, j;
while (coda_processi_pronti[k].primo == -1) { // -1 se l'elemento è l'ultimo (o la coda è vuota)
		k++;
	}
j = Prelievo(coda_processi_pronti[k]);
processo_in_esecuzione = j;
}
```
- **Ripristino_stato**, prevede il caricamento del contesto del nuovo processo nei registri del processore.
```C
void Ripristino_stato() {
int j;
j = processo_in_esecuzione;
<registro-temp> = descrittori[j].servizio.delta_t;
<registri-CPU> = descrittori[j].contesto;
}
```
Dunque il meccanismo di **cambio di contesto** si presenta come segue:
```C
void Cambio_di_Contesto() {
	int j, k;
	Salvataggio_stato();
	j = processo_in_esecuzione;
	k = descrittori[j].servizio.priorità;
	Inserimento(j, coda_processi_pronti[k]);
	Assegnazione_CPU();
	Ripristino_stato();
}
```
NB: per consentire la modalità di servizio a divisione di tempo è necessario che il nucleo gestisca un *dispositivo temporizzatore*, che ad intervalli prefissati esegua il cambio di contesto.

### 3. Implementazione del Semaforo in Sistemi Monoprocessore

Nel nucleo di un sistema monoprocessore, il semaforo può essere implementato tramite una *variabile intera* (≥0) ed un *puntatore ad una coda* (FIFO) di *descrittori di processi in attesa* sul semaforo. Ipotesi: gestione dei processi basata su priorità, ovvero al semaforo viene associato un insieme di code (una per priorità).
```C
typedef struct {
	int contatore;
	coda_a_livelli coda;
} descr_semaforo;

descr_semaforo semafori[N_max_semafori];

typedef int semaforo; // ID del semaforo nella lista 'semafori'

void P(semaforo s) {
	int j, k;
	if (semafori[s].contatore == 0) {
		Salvataggio_stato();
		j = processo_in_esecuzione;
		k = descrittori[j].servizio.priorità;
		Inserimento(j, semafori[s].coda[k]);
		Assegnazione_CPU();
		Ripristino_stato();
	}
	else semafori[s].contatore--;
}

void V(semaforo s) {
	int j, k, p, q = 0; // j, k: processi; p, q: indici priorità
	while (semafori[s].coda[q].primo == -1 && q < min_priorità)
		q++;
	if (semafori[s].coda[q].primo != -1) {
		k = Prelievo(semafori[s].coda[q];
		j = processo_in_esecuzione;
		p = descrittori[j].servizio.priorità;
		if (p < q) // il processo in esecuzione è prioritario
			Inserimento(k, coda_processi_pronti[q]);
		else { // preemption
			Salvataggio_stato();
			Inserimento(j, coda_processi_pronti[p]);
			processo_in_esecuzione = k;
			Ripristino_stato();
		}
	}
	else semafori[s].contatore++;
}
```

### 4. Spiegare le Possibili Realizzazioni del Nucleo in Sistemi Multiprocessore (SMP e Nuclei Distinti) e Confrontarle

Il SO che esegue in un'architettura multiprocessore deve gestire una molteplicità di CPU, ognuna delle quali può accedere alla stessa memoria condivisa. Vi sono 2 modelli: *SMP* e a *Nuclei Distinti*.

##### Modello SMP
Nel modello SMP (Symmetric Multi Processing) vi è un'<ins>unica copia del nucleo</ins> del Sistema Operativo, allocata <ins>nella memoria comune</ins>, che si occupa di tutte le risorse disponibili, comprese le CPU. Ogni processo può essere allocato su una qualunque CPU. È possibile che processi che eseguono su CPU diverse richiedano contemporaneamente funzioni del nucleo, ovvero vi è *competizione tra CPU* nell'esecuzione del nucleo, dunque vi è <ins>necessità di sincronizzazione</ins>.
Soluzioni:
- **Un solo lock**, ovvero viene associato al nucleo un lock per garantire la mutua esclusione delle funzioni del nucleo da parte di processi diversi. Tuttavia, in questo modo si <ins>limita il grado di parallelismo</ins>, escludendo a priori ogni possibilità di esecuzione contemporanea di funzioni del nucleo, che operano su strutture dati distinte (es: due semafori diversi).
- **Lock multipli**, ovvero si individuano all'interno del nucleo diverse classi di sezioni critiche, ognuna associata ad una struttura dati separata e sufficientemente indipendente dalle altre (es: coda processi pronti, singoli semafori), e a ciascuna viene associato un lock distinto. In questo modo si <ins>incrementa il grado di parallelismo</ins>.

Il modello SMP consente il <ins>load balancing</ins>, permettendo di <ins>schedulare equamente i processi su processori diversi</ins>. Tuttavia, in alcuni casi può essere vantaggioso assegnare un processo ad un determinato processore (usando la memoria privata del processore, in quanto se questa contiene già il codice del processo, il ripristino del contesto diventa più rapido), richiedendo però in questo caso una *coda dei processi pronti per nodo*, invece di una sola.

##### Modello a Nuclei Distinti
Il modello a nuclei distinti prevede che vi siano più istanze del nucleo, raggruppate in una collezione, che eseguono in modo concorrente. Secondo questo modello, i processi che eseguono si possono dividere fra <ins>più nodi virtuali con poche interazioni reciproche</ins>. Ogni nodo virtuale è mappato su un nodo fisico (tutte le strutture del nucleo relative al *nodo virtuale* sono allocate nella memoria privata del nodo fisico) e tutte le interazioni locali ad un nodo virtuale possono essere eseguite indipendentemente e concorrentemente a quelle locali degli altri nodi. La memoria comune dell'architettura viene utilizzata solo per permettere a processi di nodi virtuali diversi di interagire.<br/>
Nel modello a kernel distinti <ins>un processo può essere schedulato solo sul nodo contenente il relativo descrittore</ins>, rendendo impossibile l'attuazione di politiche di load balancing.

##### Confronto SMP e Nuclei Distinti
**SMP**:
- *Vantaggi*: permette una <ins>gestione ottimale delle risorse computazionali</ins>, in quanto consente il bilanciamento del carico fra le CPU dei vari nodi. Infatti, secondo questo modello lo scheduler può decidere su quale CPU (fra tutte) allocare un processo.
- *Svantaggi*: il grado di parallelismo tra CPU è sfavorito.

**Nuclei Distinti**:
- *Vantaggi*: favorisce il <ins>grado di parallelismo tra CPU</ins>, in quanto il grado di accoppiamento tra queste è minore. Ciò rende questo modello <ins>più scalabile</ins>.
- *Svantaggi*: vincola ogni processo ad essere schedulato sempre sullo stesso nodo, impedendo il bilanciamento di carico.

### 5. Implementazione del Semaforo in Sistemi Multiprocessore + Implementazione del Semaforo, delle Relative Operazioni ed il Meccanismo di Segnalazione tra i Nuclei nel caso di Context Switch
<!-- + Esempio di Interazione chiesto dalla prof -->

##### Modello SMP
In SMP i semafori sono realizzati proteggendo gli accessi ai contatori e alla coda dei processi pronti mediante lock. Se si usa un lock per ogni risorsa, due operazioni <ins>P su semafori diversi possono operare contemporaneamente solo se non sono sospensive</ins>, in quanto i semafori hanno *lock diversi*, ma la *coda dei processi pronti* è una risorsa *condivisa*, altrimenti devono operare in sequenza.<br/>
Esempio: se vi è scheduling pre-emptive con priorità, una V può portare in esecuzione un processo con priorità superiore a quella di uno dei tanti in esecuzione (anche in altre CPU). Dunque, occorre che il nucleo revochi l'accesso alla CPU di uno di questi ultimi e la assegni al processo più prioritario appena riattivato. È quindi necessario che il nucleo mantenga l'informazione del processo a più bassa priorità in esecuzione e su quale CPU esso operi, rendendo inoltre necessario l'invio di interrupt HW alle varie CPU.

##### Modello a Nuclei Distinti
Nel modello a Nuclei Distinti, poiché solo le interazioni tra processi appartenenti a nodi virtuali diversi utilizzano la memoria comune, si distinguono i semafori tra:
- **semafori privati** di un nodo virtuale, utilizzati solo dai <ins>processi appartenenti al nodo</ins>;
- **semafori condivisi** tra nodi virtuali, utilizzati da processi appartenenti a nodi diversi, e le cui <ins>informazioni</ins> sono contenute <ins>in memoria comune</ins>.

Ogni semaforo condiviso è rappresentato come:
- un <ins>intero in memoria comune</ins>, protetto da un lock;
- una <ins>coda locale per ogni nodo</ins>, contenente i descrittori dei processi locali sospesi nel semaforo;
- una <ins>coda globale di tutti i *rappresentanti* dei processi sospesi sul semaforo</ins> (il rappresentante di un processo identifica il nodo fisico su cui opera e il pid del processo).

Una P sospensiva su un semaforo condiviso porta a inserire il rappresentante del processo chiamante nella coda globale ed il descrittore nella coda locale; una V, invece, estrae un processo dalla coda globale, ne comunica l'identità al nodo virtuale relativo (tramite interruzione, per garantire il rispetto della priorità), il quale risveglia il processo estraendo il descrittore dalla propria coda locale.

Implementazione in pseudo-C:
```C
void P(semaphore s) {
	if (is_private(s)) {
		// P come nel caso monoprocessore
	} else {
		lock(s.common_lock);
		// P
		// se necessario sospende il rappresentante nel processo in s.q
		unlock(s.common_lock);
	}
}

void V(semaphore s) {
	if (is_private(s)) {
		// P come nel caso monoprocessore
	} else {
		lock(s.common_lock);
		if (!empty(s.q)) {
			if (s.node == current_node) {
				// P come nel caso monoprocessore
			} else {
				// estrae p da s.q
				int ch = get_buffer(p.node);
				while (busy(ch)) {}
				send(ch, p.id);
				interrupt(p.cpu);
			}
		} else {
			p.c++;
		}
		unlock(s.common_lock);
	}
}
```

<h1 align="center">MODELLO A SCAMBIO DI MESSAGGI</h1>

### 1. Definire le Caratteristiche del Modello a Scambio di Messaggi ed il Concetto di Canale di Comunicazione

Nel modello a scambio di messaggi:
- ogni processo può accedere esclusivamente alle <ins>risorse allocate nella propria memoria locale/privata</ins>;
- ogni risorsa del sistema è accessibile direttamente da un solo processo (<ins>gestore</ins>);
- se una risorsa è necessaria a più processi, ciascuno di questi (client) dovrà <ins>delegare l'unico processo in grado di operarvi</ins> (server/gestore), il quale restituirà successivamente i risultati;
- il meccanismo di base per qualunque tipo di interazione fra i processi è lo <ins>scambio di messaggi</ins>.

**Canale di Comunicazione**: collegamento logico mediante il quale 2 o più processi comunicano. L'astrazione del canale è realizzata dal kernel come meccanismo primitivo per lo scambio di informazioni, mentre è compito dei linguaggi di programmazione offrire gli strumenti linguistici di alto livello per istanziarli ed utilizzarli.<br/>
Caratteristiche:
1. <ins>direzione del flusso dei dati</ins> che il canale può trasferire (*monodirezionale* o *bidirezionale*);
2. <ins>designazione</ins> dei processi <ins>mittente e destinatario</ins>:
	- *link* = uno-a-uno (canale simmetrico);
	- *port* = molti-a-uno (canale asimmetrico);
	- *mailbox* = molti-a-molti (canale asimmetrico);
3. <ins>tipo di sincronizzazione</ins> fra i processi comunicanti (comunicazione *asincrona*, *sincrona* o con *sincronizzazione estesa*).

### 2. Spiegare la Differenza tra Comunicazione Asincrona, Sincrona e con Sincronizzazione Estesa

**Comunicazione Asincrona**: il processo <ins>mittente continua la sua esecuzione</ins> immediatamente dopo l'invio del messaggio.<br/>
Proprietà:
- la <ins>carenza espressiva</ins> rende difficile la verifica dei programmi, in quanto la ricezione del messaggio può avvenire in un istante successivo all'invio e, di conseguenza, il messaggio ricevuto non contiene informazioni attribuibili allo stato attuale del mittente;
- <ins>favorisce il grado di concorrenza/parallelismo</ins>, in quanto l'invio di un messaggio non costituisce un punto di sincronizzazione per mittente e destinatario;
- <ins>serve un buffer di capacità limitata</ins>, in quanto un buffer di dimensioni illimitate non è concretamente realizzabile e, per mantenere inalterata la semantica, bisogna sospendere il processo mittente se il buffer è pieno.

**Comunicazione Sincrona** (o rendez-vous semplice): <ins>il primo processo</ins> che esegue l'operazione di comunicazione (invio o ricezione) <ins>si sospende</ins>, in attesa che l'altro sia pronto ad eseguire l'operazione corrispondente.<br/>
Proprietà:
- favorisce l'<ins>espressività</ins>, in quanto l'invio di un messaggio è un punto di sincronizzazione, ed il messaggio ricevuto contiene informazioni attribuibili allo stato attuale del processo mittente (verifica dei programmi semplificata);
- il <ins>grado di parallelismo è minore</ins>, rispetto alla comunicazione asincrona;
- <ins>non servono buffer</ins>, in quanto un messaggio può essere inviato solo se il destinatario è pronto a riceverlo.

**Comunicazione con Sincronizzazione Estesa** (o rendez-vous esteso): è semanticamente analogo alla chiamata di procedura remota (<ins>RPC</ins>), in quanto ogni messaggio inviato rappresenta una richiesta al destinatario dell'esecuzione di una certa azione. Il mittente rimane in attesa dopo l'invio della richiesta e si sblocca quando riceve la risposta (con gli eventuali risultati).<br/>
Proprietà:
- <ins>sfrutta il modello client/server</ins>;
- <ins>elevata espressività</ins> (verificabilità dei programmi);
- <ins>riduzione del grado di parallelismo</ins>.

### 3. Descrivere le Primitive Send e Receive e Spiegare com'è possibile Realizzare una Send Sincrona con una Asincrona e Viceversa

La **Send** è una primitiva di comunicazione che esprime l'invio di un messaggio ad un canale identificato univocamente. Può essere <ins>asincrona</ins> (canale bufferizzato) o <ins>sincrona</ins> (buffer di capacità nulla), ovvero il processo attende che il ricevente esegua la receive prima di proseguire la propria esecuzione.

La **Receive** è una primitiva di comunicazione che esprime la lettura di un messaggio da un canale identificato univocamente, salvandone il contenuto in una variabile. È <ins>bloccante</ins> (sospende il processo che la esegue) se sul canale non ci sono messaggi da leggere.

L'istruzione di più basso livello è la send asincrona. Per implementare una send sincrona si può inviare un messaggio e rimanere in attesa, su un altro canale, di un messaggio <ins>ACK</ins>, sfruttando la semantica bloccante della receive.

Per costruire invece una send asincrona, con buffer di capacità N, da una send sincrona, è possibile utilizzare una mailbox concorrente. Ovvero, si possono utilizzare <ins>N processi concorrenti collegati in cascata</ins>, ciascuno dei quali esegue una receive dal precedente ed una send verso il successivo (il primo riceve dal processo mittente "produttore", l'ultimo invia al processo destinatario "consumatore").

### 4. Spiegare la Semantica della Receive, quali Problemi può dare e Come si Possono Risolvere

Supponiamo di avere un processo server che fornisce diversi servizi, ognuno dei quali viene attivato in seguito alla ricezione di un messaggio su un canale diverso, da parte di un processo client.

**Problema**: poiché ci sono più canali, il server deve ciclicamente eseguire una receive su ciascun canale, per verificare lo stato delle richieste. Tuttavia, poiché la receive ha semantica bloccante, il <ins>server potrebbe bloccarsi</ins> leggendo da un canale, <ins>mentre sono presenti messaggi in attesa di essere letti su altri canali</ins>.

**Soluzione**: si potrebbe realizzare una <ins>receive con semantica non bloccante</ins>. Il server, prima di eseguire la receive da un canale, ne controlla lo stato:
- se sono presenti messaggi, ne legge uno;
- altrimenti, se nel canale non sono presenti messaggi, passa al successivo.
<!-- ad esempio, in Go si potrebbe realizzare una funzione non_blocking_receive, che verifica con len() lo stato del canale e, se c'è almeno un messaggio, ovvero len() > 0, effettua la receive e resituisce il valore; altrimenti, restituisce un valore nullo oppure un errore. -->

In questo modo la receive non sospende mai il processo server, generando però un **ulteriore problema**: l'<ins>attesa attiva</ins> (se tutti i canali sono vuoti, il server continua ad iterare).

**Meccanismo di Ricezione Ideale**:
- consente al server di <ins>verificare contemporaneamente la disponibilità di messaggi su più canali</ins>;
- abilita la <ins>ricezione di un messaggio da un qualunque canale contenente messaggi</ins>;
- <ins>quando tutti i canali sono vuoti, blocca il processo in attesa che arrivi un messaggio</ins>, qualunque sia il canale su cui arriva.

Questo meccanismo è realizzabile tramite i *comandi con guardia*.

### 5. Discutere il Comando con Guardia

Il comando con guardia permette di realizzare un meccanismo di ricezione ideale.<br/>
Sintassi: ```<guardia> -> <istruzione>;```<br/>
dove ```<guardia>``` è costituita dalla coppia ```(<espressione_booleana> ; <receive>)```.<br/>
L'espressione boooleana viene detta **guardia logica**, mentre l'operazione di receive viene detta **guardia d'ingresso** ed ha semantica <ins>bloccante</ins>.

La valutazione di una guardia può fornire 3 diversi valori:
1. **guardia fallita**, se l'espressione booleana ha il valore <ins>false</ins>;
2. **guardia ritardata**, se l'espressione booleana ha valore <ins>true</ins> e nel canale su cui viene eseguita <ins>non ci sono messaggi</ins>;
3. **guardia verificata**, se l'espressione booleana ha valore <ins>true</ins> e nel canale <ins>c'è almeno un messaggio</ins> (dunque la receive può essere eseguita subito).

L'esecuzione di un comando con guardia determina un effetto diverso in base alla valutazione della guardia:
1. in caso di *guardia fallita*, il comando termina senza produrre alcun effetto;
2. in caso di *guardia ritardata*, il processo si sospende finché non arriva un messaggio sul canale, dopodiché verrà eseguita la receive e successivamente l'istruzione;
3. in caso di *guardia valida*, il processo esegue la receive e successivamente l'istruzione.

**Comando con Guardia Alternativo** (`select`): racchiude un numero arbitrario di comandi con guardia semplici. Esso valuta le guardie di tutti i rami e si possono verificare 3 casi:
1. se *tutte le guardie sono fallite*, il comando termina;
2. se *tutte le guardie non fallite sono ritardate*, il processo in esecuzione si sospende in attesa che arrivi un messaggio, dopodiché verrà eseguita la receive relativa e successivamente l'istruzione;
3. se *una o più guardie sono valide*: 
 - viene scelto in modo <ins>non deterministico</ins> uno dei rami con guardia valida;
 - viene eseguita la relativa receive;
 - viene eseguita successivamente l'istruzione;
 - l'esecuzione dell'intero comando alternativo termina.

NB: la scelta del ramo fra quelli con guardia valida è non deterministica per non imporre una politica preferenziale tra i vari casi.

Sintassi del comando con guardia alternativo:
```C
select {
[ ] <guardia_1> -> <istruzione_1>;
[ ] <guardia_2> -> <istruzione_2>;
...
[ ] <guardia_n> -> <istruzione_n>;
}
```

**Comando con Guardia Ripetitivo**: ha un comportamento analogo al caso "alternativo", ma il ciclo ricomincia tutte le volte che viene eseguita un'istruzione, terminando solo se tutte le guardie sono fallite.

Sintassi del comando con guardia ripetitivo:
```C
do {
[ ] <guardia_1> -> <istruzione_1>;
...
[ ] <guardia_n> -> <istruzione_n>;
}
```

<h1 align="center">COMUNICAZIONE CON SINCRONIZZAZIONE ESTESA</h1>

### 1. Spiegare in Cosa Consiste la Comunicazione con Sincronizzazione Estesa

La sincronizzazione estesa è un meccanismo di comunicazione che prevede che un processo chiamante richieda un servizio al processo destinatario e rimanga sospeso fino al completamento del servizio richiesto. Entrambi i processi rimangono sincronizzati durante l'esecuzione del servizio, fino a quando il mittente non riceve i risultati da parte del ricevente.

Semanticamente, la sincronizzazione estesa è <ins>analoga ad una chiamata di funzione</ins>, in quanto il programma chiamante prosegue solo dopo che l'esecuzione della funzione è stata completata dal ricevente. 

La differenza sostanziale è che il servizio richiesto viene eseguito remotamente da un processo differente da quello chiamante. Il server può essere implementato in 2 modi: *Remote Procedure Call* (RPC) oppure *rendez-vous esteso*.

### 2. Descrivere RPC e Rendez-Vous Esteso e Confrontarli

##### RPC
Per ogni operazione che il client può richiedere viene dichiarata <ins>una procedura</ins> lato server. Al momento dell'effettiva richiesta, il <ins>server crea un nuovo processo tramite **fork** (thread)</ins>, il cui compito è di effettuare una chiamata alla procedura corrispondente e, una volta terminata l'operazione, <ins>invia direttamente lui stesso la risposta</ins> al client.<br/>
L'insieme delle procedure remote è definito all'interno di un componente software (*modulo*), che contiene anche le variabili locali al server, ed eventuali procedure e processi locali. I singoli moduli operano in spazi di indirizzamento diversi e possono quindi essere allocati su nodi distinti di una rete.

##### Rendez-Vous Esteso
Ogni operazione viene specificata come un insieme di istruzioni, preceduto da un'<ins>istruzione **accept** che sospende il processo server</ins> (sincronizzazione) in attesa di una richiesta dell'operazione. All'arrivo della richiesta il processo esegue il relativo insieme di istruzioni e i risultati ottenuti sono inviati al chiamante.<br/>
La accept è bloccante se non sono presenti richieste di servizio. Se uno stesso servizio viene richiesto da più processi, le richieste vengono inserite in una coda associata al servizio, gestita con politica FIFO. Ad uno stesso servizio possono essere associate più accept nel codice eseguito dal server, dunque <ins>ad una richiesta possono corrispondere azioni diverse</ins>. Lo schema di comunicazione realizzato dal meccanismo di rendez-vous è di tipo asimmetrico molti-a-uno.<br/>
Il server può selezionare le richieste da servire in base al suo <ins>stato interno</ins>, utilizzando i comandi con guardia; oppure anche in base ai <ins>parametri di ingresso della richiesta</ins>, anche in questo caso specificando i controlli da effettuare nel comando con guardia.<br/>
Ada è un linguaggio molto espressivo dal punto di vista della comunicazione fra processi, perché permette, ad esempio, di eseguire operazioni diverse (accept diverse) per una richiesta dello stesso tipo.

##### Differenze
- RPC rappresenta solo un meccanismo di <ins>comunicazione</ins> fra processi, mentre delega al programmatore la gestione della sincronizzazione dei vari processi figli del servitore (tramite utilizzo di monitor, semafori), permettendo di eseguire più operazioni concorrentemente (es: Java RMI, Distributed Processes).
- Rendez-vous Esteso combina <ins>comunicazione con sincronizzazione</ins>, in quanto vi è un solo processo server, al cui interno sono definite le istruzioni che consentono di realizzare il servizio richiesto.

<h1 align="center">ALGORITMI DI SINCRONIZZAZIONE DISTRIBUITI</h1>

### 1. Caratteristiche di un Sistema Distribuito, Proprietà Desiderate e Sincronizzazione

In un sistema distribuito i processi eseguono su nodi fisicamente separati, collegati tra loro da una rete di interconnessione, ed il modello a scambio di messaggi è la sua naturale astrazione.<br/>
Caratteristiche: concorrenza/parallelismo delle attività dei nodi; assenza di risorse condivise tra nodi; assenza di un clock globale; possibilità di malfunzionamenti indipendenti dei nodi (crash, attacchi, ...), o della rete di comunicazione (latenza, packet loss).

**Proprietà Desiderate**:
- **scalabilità**, nell'applicazione distribuita le prestazioni dovrebbero crescere al crescere del numero di nodi utilizzati;
- **tolleranza ai guasti**, l'applicazione dev'essere in grado di funzionare anche in presenza di guasti (crash dei nodi, problemi di rete, ...).

**Speedup**: indicatore per misurare le *prestazioni* di un sistema parallelo/distribuito. Lo speedup per N nodi è dato dal rapporto tra il tempo di esecuzione dell'applicazione ottenuto con un solo nodo e quello ottenuti con N nodi, ovvero: ```speedup(N) = tempo(1) / tempo(N)```. Il caso ideale (sistema scalabile al 100%) è ```speedup(N) = N```.

**Tolleranza ai Guasti**: un sistema distribuito si dice tollerante ai guasti se riesce ad *erogare i propri servizi anche in presenza di guasti* (temporanei, intermittenti o persistenti) in uno o più nodi. Un sistema tollerante ai guasti deve nascondere i problemi agli altri processi, ad esempio tramite ridondanza.

**Algoritmi di Sincronizzazione**: come nel modello a memoria comune, anche nel modello a scambio di messaggi è importante poter disporre di algoritmi di sincronizzazione tra i processi concorrenti, che permettano di <ins>coordinare opportunamente i vari processi</ins>:
- *timing*, sincronizzazione dei clock e tempo logico;
- *mutua esclusione* distribuita;
- *elezione di coordinatori* di gruppi di processi.

In ogni caso, è sempre desiderabile che tali algoritmi godano di scalabilità e tolleranza ai guasti (e vengono valutati anche in base a tali parametri).

### 2. Spiegare il Problema della Gestione del Tempo nei Sistemi Distribuiti ed una Possibile Soluzione

In un sistema distribuito gli orologi di ogni nodo <ins>non sempre sono sincronizzati</ins>, dunque è possibile che l'ordine nel quale due eventi vengono registrati sia diverso da quello in cui sono effettivamente accaduti, e questo può generare problemi.<br/>
Per questo motivo, gli orologi utilizzati in applicazioni distribuite si dividono in **fisici**, che forniscono l'ora esatta, e **logici**, che permettono di associare un <ins>timestamp</ins> coerente con l'ordine in cui gli eventi si sono effettivamente verificati.

**Orologi Logici**: per implementare gli orologi logici, si definisce una relazione *happened-before* "→", tale che:
1. *A* e *B* sono eventi in uno stesso processo ed *A* si verifica prima di *B*, allora *A* → *B*;
2. *A* è l'evento di invio di un messaggio e *B* è l'evento di ricezione dello stesso, allora *A* → *B*;
3. vale la proprietà transitiva, ovvero se *A* → *B*, e *B* → *C*, allora *A* → *C*.

Assumiamo quindi che ad ogni evento *e* venga associato un timestamp *C(e)* e che tutti i processi concordino su questo, per cui vale la proprietà **[*]**: ```A → B ⟺ C(A) < C(B)```. Dunque, se all'interno di un processo *A* precede *B*, avremo che *C(A)* < *C(B)*; se *A* è l'evento di invio e *B* l'evento di ricezione dello stesso messaggio, allora *C(A)* < *C(B)*.

##### Algoritmo di Lamport
L'algoritmo di *Lamport* fornisce una soluzione al problema della sincronizzazione dei processi in un contesto distribuito, basata sull'utilizzo di orologi logici, implementati tramite timestamp. In particolare, per garantire il rispetto della proprietà [*], l'algoritmo afferma che:
1. ogni processo *Pi* gestisce localmente un <int>contatore</int> *Ci* del tempo logico;
2. ogni evento del processo fa incrementare il contatore di 1 (*Ci*++);
3. ogni volta che il processo Pi invia un messaggio *m*, il contatore viene incrementato (*Ci*++) e successivamente al messaggio viene assegnato il timestamp *ts*(*m*)=*Ci*;
4. quando un processo *Pj* riceve un messaggio *m*, assegna al proprio contatore *Cj* un valore dato dal massimo tra *Cj* e *ts*(*m*), ovvero ```Cj = max{Cj, ts(m)}```, e successivamente lo incrementa di 1 (*Cj*++).

### 3. Algoritmi di Sincronizzazione Distribuiti: Mutua Esclusione e Soluzioni Possibili (PRO e CONTRO)

Nei sistemi distribuiti è spesso necessario garantire che due processi non possano eseguire contemporaneamente alcune attività, ad esempio quelle che prevedono accesso a risorse condivise. Questo problema può essere risolto in maniera:
- *centralizzata*, delegando la gestione ad un processo <ins>coordinatore</ins> al quale tutti gli altri processi si rivolgono per l'utilizzo della risorsa;
- *decentralizzata*, sincronizzando i prrocessi mediante algoritmi la cui logica è distribuita tra i processi stessi (questo è generalmente un approccio più scalabile, in quanto avere un coordinatore singolo costituisce un collo di bottiglia).

Le soluzioni al problema della mutua esclusione distribuita si dividono inoltre in:
- *permission-based* (centralizzati o decentralizzati), nelle quali ogni processo che vuole eseguire la sezione critica (operazione mutuamente esclusiva) "<ins>richiede il permesso</ins>" di eseguire ad uno o più altri processi;
- *token-based* (sempre decentralizzati), in cui i processi si passano un <ins>token</ins> che concede l'autorizzazione ad eseguire la propria sezione critica.

##### Soluzione Centralizzata
La soluzione *centralizzata* prevede la presenza di un processo coordinatore che espone due primitive di <ins>richiesta</ins> e <ins>rilascio</ins> della risorsa. Ogni processo che vuole eseguire la propria sezione critica si rivolge al coordinatore per ottenere il *permesso*. Il coordinatore gestisce una richiesta alla volta, utilizzando una <ins>coda FIFO</ins>: se un processo richiede una risorsa che è attualmente utilizzata da un altro processo, viene messo in attesa in una coda, e risvegliato dal coordinatore stesso, quando la risorsa si libera di nuovo (ed è il suo turno nella coda).
- **Vantaggi**: è un <ins>algoritmo equo</ins> (è privo di *starvation*), ed è implementabile utilizzando <ins>solo 3 messaggi</ins> (richiesta, autorizzazione, rilascio) per ciascuna sezione critica.
- **Svantaggi**: è <ins>poco scalabile</ins>, in quanto al crescere del numero dei nodi il *coordinatore* può diventare un *collo di bottiglia*; è <ins>poco tollerante ai guasti</ins> e prevede un *Single Point of Failure*, in quanto se si guasta il coordinatore, l'intero sistema si blocca, e inolte, se un processo non ottiene una risposta, non può distinguere il motivo (autorizzazione non concessa o guasto).

##### Algoritmo di Ricart-Agrawala
L'algoritmo di *Ricart-Agrawala* è una soluzione *decentralizzata permission-based* che richiede, come requisito per il suo funzionamento, la presenza di un <ins>orologio logico sincronizzato (timestamp)</ins>. Ad ogni processo sono associati 2 thread concorrenti: **main**, che esegue la sezione critica, e **receiver** che riceve le autorizzazioni.

**Main**: quando un main vuole entrare nella sezione critica:
1. manda una ```RICHIESTA``` d'autorizzazione (con il proprio PID e timestamp) a tutti gli altri nodi;
2. attende le autorizzazioni (```OK```) di tutti gli altri nodi;
3. esegue la sezione critica;
4. invia un ```OK``` a tutte le richieste in attesa.

**Receiver**: quando un receiver riceve una richiesta, esso può trovarsi in 3 possibili stati:
1. **RELEASED**, se il processo non è interessato ad eseguire la sezione critica (ed il proprio main non ha inviato richieste), dunque <ins>risponde ```OK```</ins>;
2. **WANTED**, se il processo vuole entrare nella sezione critica (dunque il proprio main è in attesa dell'autorizzazione ```OK```), allora <inst>confronta il timestamp</ins> della richiesta ricevuta *Tr* con quello della richiesta inviata *Ts*:
	- se *Tr* < *Ts*, <ins>risponde con ```OK```</ins>;
	- altrimenti (*Tr* ≥ *Ts*), non risponde e <ins>mette la richiesta ricevuta in coda</ins>;
3. **HELD**, se sta eseguendo la sezione critica, nel qual caso <ins>la richiesta viene messa in coda</ins>.

- **Vantaggi**: è <ins>molto scalabile</ins>.
- **Svantaggi**: ha un <ins>maggiore costo di comunicazione</ins> per singolo partecipante, in quanto sono necessari 2\*(N-1) messaggi per ciascuna sezione critica (il processo in stato *WANTED* invia ```RICHIESTA``` e riceve ```OK``` da parte di tutti gli altri nodi); presenta <ins>poca tolleranza ai guasti</ins> in quanto presenta *N Points of Failure*, in quanto se un nodo va in crash, questo non risponderà più alle richieste, facendo rimanere i processi in attesa.

**Soluzione al Problema dei Guasti**: si può modificare il protocollo, prevedendo un messaggio dopo l'invio della risposta:
- ```OK```, in caso di autorizzazione;
- ```ATTESA```, in caso il processo opposto si trovi in stato di *HELD*.

In questo modo, basterà impostare un <ins>timeout</ins> nel richiedente per rilevare la presenza di guasti nel destinatario.

##### Algoritmo Token-Ring
L'algoritmo *Token-Ring* è una soluzione *decentralizzata token-based* che prevede che i processi siano collegati tra di loro secondo una <ins>topologia ad anello orientato</ins>, in cui ciascun processo conosce i suoi vicini, e si scambiano un messaggio (token) nel verso relativo all'ordine dei processi. Il token rappresenta il permesso unico di eseguire sezioni critiche.<br/>
Quando un processo riceve il token:
1. se si trova in stato **WANTED**, allora <ins>trattiene il token</ins> ed esegue la propria sezione critica, dopodiché (una volta terminata l'operazione) passa il token al processo successivo;
2. se si trova in stato **RELEASED**, <ins>passa direttamente il token</ins> al processo successivo nell'anello.

- **Vantaggi**: è <ins>molto scalabile</ins>;
- **Svantaggi**: ha un <ins>costo di comunicazione variabile</ins> (il numero di messaggi per ogni sezione critica dipende dal numero dei nodi presenti, dunque è *potenzialmente infinito*); come per Ricart-Agrawala, <ins>non è tollerante ai guasti</ins> e presenta *N Points of Failure* e vi è la possibilità di perdere il token se il nodo che lo detiene va in crash.

**Soluzione al Problema dei Guasti**: come per Ricart-Agrawala, si può modificare il protocollo per prevedere che ad ogni invio del token, venga restituita una <ins>risposta</ins> e, in caso questa non arrivi entro un <ins>timeout</ins>, il nodo viene considerato guasto, escluso dall'anello e si passa il token al successivo.

### 4. Algoritmi di Sincronizzazione Distribuiti: Elezione del Coordinatore
 
In alcuni algoritmi è previsto che un processo **coordinatore** rivesta un ruolo speciale nella sincronizzazione tra i vari nodi. La designazione del coordinatore può essere *statica*, se viene scelto prima dell'esecuzione, o *dinamica*, mediante un <ins>algoritmo di elezione</ins> a tempo di esecuzione. Quest'ultima permette, di cambiare coordinatore a runtime se quello attuale smette di rispondere (ad esempio a causa di un guasto).<br/>
**Assunzioni di base**: ogni processo è identificato da un ID univoco; ogni processo conosce gli ID di tutti gli altri (ma non il loro stato).
**Obbiettivo**: viene designato vincitore (nuovo coordinatore) il processo attivo con l'ID più alto.

##### Algoritmo Bully
L'algoritmo di elezione *Bully* prevede che quando un processo *Pk* (k = 1, ..., N) rileva che il coordinatore non è più attivo, organizzi un'elezione:
1. *Pk* invia un messaggio ```ELEZIONE``` a tutti i processi con ID più alto del suo;
2. se nessun processo risponde, *Pk* vince l'elezione e diventa il nuovo coordinatore, dunque comunica a tutti gli altri il nuovo ruolo inviando un messaggio ```COORDINATORE```;
3. se un processo *Pj* (j > k) risponde, *Pj* prende il controllo dell'elezione, e *Pk* rinuncia, smettendo di rispondere ai successivi messaggi ```ELEZIONE```. 

Ogni processo attivo risponde ad ogni messaggio ```ELEZIONE``` ricevuto.

##### Algoritmo ad Anello
L'algoritmo di elezione ad *Anello* prevede che i processi siano collegati tramite una topologia logica ad anello orientato, in cui i processi sono posizionati in ordine in base al loro ID, che rappresenta anche la loro priorità. Quando un processo *Pk* rileva che il coordinatore non è più attivo (non risponde), organizza un'elezione:
1. *Pk* invia un messaggio ```ELEZIONE``` contenente il suo ID al successore, bypassandolo in caso sia in crash (si presuppone che un processo abbia gli strumenti per farlo);
2. quando un processo *Pi* riceve un messaggio ```ELEZIONE```:
	- se il messaggio non contiene il suo ID (di *Pj*), aggiunge il suo ID al messaggio e lo spedisce al successivo;
	- se il messaggio contiene il suo ID, significa che è stato compiuto un <ins>giro completo dell'anello</ins>, dunque *Pj* designa come coordinatore il processo avente l'ID più alto nel messaggio, e invia al successivo un messaggio ```COORDINATORE```, contenente l'ID del processo designato come nuovo coordinatore;
3. quando un processo riceve un messaggio ```COORDINATORE```, notifica il risultato dell'elezione al successivo, che farà lo stesso con quello dopo, e così via.

### 5. Spiegare degli Esempi forniti dalla Prof

##### Esempio 1: Algoritmo di Ricart-Agrawala
Abbiamo 5 processi:
- *P1* in stato **HELD**;
- *P2* in **RELEASED**;
- *P3* in **WANTED** con **```ts(m) = 3```**;
- *P4* in **RELEASED**;
- *P5* in **WANTED** con **```ts(m) = 5```**.

Spiegare come e in quale ordine vengono gestite le richieste, secondo l'algoritmo di *Ricart-Agrawala*.

<table>
	<tr>
		<td width="5%" align="center"><b>Stato</b></td>
		<td width="19%" align="center"><b><i>P1</i></b></td>
		<td width="19%" align="center"><b><i>P2</i></b></td>
		<td width="19%" align="center"><b><i>P3</i></b></td>
		<td width="19%" align="center"><b><i>P4</i></b></td>
		<td width="19%" align="center"><b><i>P5</i></b></td>
	</tr>
	<tr>
		<td align="center">(1)</td>
		<td align="center"><b>HELD</b><br/>sta eseguendo la propria sezione critica</td>
		<td align="center"><b>RELEASED</b></td>
		<td align="center"><b>WANTED</b><br/>
			<ins>invia</ins> <code>RICHIESTA</code> con <b><code>ts(m) = 3</code></b></td>
		<td align="center"><b>RELEASED</b></td>
		<td align="center"><b>WANTED</b><br/>
			<ins>invia</ins> <code>RICHIESTA</code> con <b><code>ts(m) = 5</code></b></td>
	</tr>
	<tr>
		<td align="center">(2)</td>
		<td align="center"><b>HELD</b><br/>
			riceve le richieste di <i>P3</i> e <i>P5</i> e le <ins>mette in coda</ins></td>
		<td align="center"><b>RELEASED</b><br/>
			riceve le richieste di <i>P3</i> e <i>P5</i> e <ins>risponde</ins> <code>OK</code> a entrambi</td>
		<td align="center">
			<b>WANTED <code>ts(m) = 3</code></b><br/>
			1) riceve <code>OK</code> da <i>P2</i>, <i>P4</i>, <i>P5</i><br/>
			2) riceve la richiesta di <i>P5</i> e la <ins>mette in coda</ins> poiché 3 < 5
		</td>
		<td align="center"><b>RELEASED</b><br/>
			riceve le richieste di <i>P3</i> e <i>P5</i> e <ins>risponde</ins> <code>OK</code> a entrambi</td>
		<td align="center">
			<b>WANTED <code>ts(m) = 5</code></b><br/>
			1) riceve <code>OK</code> da <i>P2</i>, <i>P4</i><br/>
			2) riceve la richiesta di <i>P3</i> e <ins>risponde</ins> <code>OK</code> poiché 3 < 5
		</td>
	</tr>
	<tr>
		<td align="center">(3)</td>
		<td align="center"><b>RELEASED</b><br/><ins>estrae dalla coda</ins> tutti i processi e <ins>invia</ins> <code>OK</code> a <i>P3</i> e <i>P5</i></td>
		<td align="center"><b>RELEASED</b></td>
		<td align="center">
			<b>HELD</b><br/>
			1) riceve <code>OK</code> da <i>P1</i><br/>
			2) ha ottenuto tutte le autorizzazioni, entra in sezione critica
		</td>
		<td align="center"><b>RELEASED</b></td>
		<td align="center">
			<b>WANTED</b><br/>
			1) riceve <code>OK</code> da <i>P1</i><br/>
			2) gli manca ancora l'<code>OK</code> di <i>P3</i>
		</td>
	</tr>
	<tr>
		<td align="center">(4)</td>
		<td align="center"><b>RELEASED</b></td>
		<td align="center"><b>RELEASED</b></td>
		<td align="center"><b>RELEASED</b><br/><ins>estrae dalla coda</ins> tutti i processi e <ins>invia</ins> <code>OK</code> a <i>P5</i>
		</td>
		<td align="center"><b>RELEASED</b></td>
		<td align="center">
			<b>HELD</b><br/>
			1) riceve <code>OK</code> da <i>P3</i><br/>
			2) ha ottenuto tutte le autorizzazioni, entra in sezione critica
		</td>
	</tr>
</table>

<h1 align="center">HIGH PERFORMANCE COMPUTING (HPC)</h1>

### 1. Evoluzione delle Architetture fino al Modello Distribuito

La principale motivazione per lo sviluppo delle tecnologie HW e SW per il parallel computing è l'<ins>aumento di performance</ins>: risolvere problemi di complessità elevata in tempi contenuti; risolvere gli stessi problemi in tempi più bassi.

**Evoluzione delle Architetture**: fino ai primi anni 2000 l'evoluzione dei sistemi di calcolo è stata "governata" dalla *Legge di Moore*, secondo cui le performance dei processori crescono costantemente (raddoppiando la densità di transistor all'interno dei chip ogni 18 mesi, con conseguente aumento di *capacità di elaborazione* del chip e aumento della *velocità di calcolo*). A partire dai primi anni 2000, ci si è trovati sempre più prossimi ai <ins>limiti fisici</ins> dei componenti: a causa dell'*effetto joule* (lega la produzione di calore al passaggio di corrente elettrica nei circuiti integrati), non è stato più possibile ad esempio aumentare la frequenza di clock, rendendo <ins>necessario l'aumento di capacità di calcolo a parità di frequenza</ins>. Questo obbiettivo è stato raggiunto grazie all'introduzione di diverse forme di parallelismo a livello HW. Infatti, se l'HW è in grado di svolgere più operazioni per ciclo, la velocità di elaborazione dell'intero sistema aumenta (più processori su singolo chip, più processori su più chip).

### 2. Cos'è il Von Neumann Bottleneck e Come si può Mitigare. Quali Architetture della Tassonomia di Flynn possono superare questo problema? 

Il modello di Von Neumann descrive lo schema funzionale di un tradizionale sistema sequenziale. L'unica CPU è collegata alla memoria centrale da un mezzo di interconnessione (es: bus), e questa separazione costituisce una limitazione nella velocità di accesso a dati e istruzioni, che influisce sulla velocità di elaborazione del sistema.

**Von Neumann Bottleneck**: il <ins>bus che collega CPU e memoria centrale</ins> costituisce un collo di bottiglia. Infatti, questo limita la velocità di fetching di istruzioni e dati, che dipendono dalla velocità del bus, e limita conseguentemente la velocità di esecuzione.<br/>
Per mitigare questo problema, è stato introdotto l'utilizzo di:
- memorie *cache*;
- *parallelismo di basso livello* (*ILP* e *HW multithreading*).

**Cache**: è una memoria *associativa* ad <ins>accesso veloce</ins>, in quanto risiede sul chip del processore e si colloca ad un livello intermedio tra memoria centrale e registri del processore; di <ins>capacità limitata</ins>, in quanto non può contenere tutte le istruzioni ed i dati necessari al programma in esecuzione.<br/>Viene gestita con criteri basati sul [*Principio di Località*](https://it.wikipedia.org/wiki/Principio_di_localit%C3%A0_(informatica)), secondo cui: "durante l'esecuzione di una data istruzione presente in memoria, con molta probabilità le successive istruzioni saranno ubicate nelle vicinanze di quella in corso" (località spaziale e/o temporale).<br/>
Dunque, si potranno avere dei *cache hit*, se l'informazione richiesta è presente in cache, oppure *cache miss*, se non è presente e va caricata dalla memoria centrale. Se la gestione è tale da mantenere un hit-rate sufficientemente elevato, gli effetti del Von Neumann Bottleneck possono essere mitigati.

**Parallelismo di Basso Livello (ILP)**: le istruzioni per essere eseguite seguono una sequenza di fasi (fetching operandi, confronto esponenti e/o shift, somma, normalizzazione risultato e memorizzazione del risultato). Ciascuna di queste può essere separata ed affidata ad un'<ins>unità funzionale</ins> indipendente che opera in parallelo alle altre. Le unità funzionali sono collegate tra di loro mediante una <ins>pipeline</ins> <br/>
Problema: non sempre questa operazione è fattibile, ad esempio se in un programma è presente una lunga serie di istruzioni tra loro dipendenti (tipo la Serie di Fibonacci).

**HW Multithreading**: i processori moderni offrono parallelismo di alto livello (a livello di thread), mediante HW multithreading, che permette a più thread di condividere la stessa CPU usando una <ins>tecnica di sovrapposizione</ins> (duplicazione registri e context switch efficiente con supporto HW). Esistono 2 approcci:
- **multithreading a grana fine** (fine-grained), secondo cui viene eseguito <ins>un context switch dopo ogni istruzione</ins>.
	- *Svantaggio*: velocità thread bassa;
	- *Vantaggio*: throughtput alto (ovvero si trasmettono più dati);
- **multithreading a grana grossa** (coarse-grained), secondo cui <ins>il context switch avviene quando il thread corrente si trova in attesa</ins> (es: attesa del caricamento di informazioni dalla memoria centrale in seguito ad un cache miss).
	- *Vantaggio*: velocità thread alta;
	- *Svantaggio*: throughtput basso.

ILP e HW Multithreading hanno permesso un miglioramento delle prestazioni dei processori, tuttavia tali meccanismi sono trasparenti ai programmatori (Modello *Von Neumann Esteso*). Nei sistemi HPC, invece, il parallelismo disponibile è visibile ai programmatori, che progetta il software sfruttando al meglio le risorse computazionali: **architetture non Von Neumann**.

Nella *Tassonomia di Flynn*, i sistemi HPC riguardano le classi SIMD e MIMD. In particolare le architetture MIMD prevedono l'asincronicità delle attività nei diversi nodi, permettendo ad ogni CPU di eseguire una sequenza di istruzioni diversa dagli altri nodi. Sistemi HPC si dividono in due modelli: a Shared Memory o Distributed Memory. La maggior parte dei sistemi HPC al giorno d'oggi presenta un <ins>modello ibrido</ins> che combina il modello a *memoria distribuita* col modello a *memoria comune*.

### 3. Quali sono le Metriche per Valutare le Prestazioni di Applicazioni Parallele. Qual è la Differenza Tra Scalabilità Weak e Scalabilità Strong

Per misurare le prestazioni di un sistema HPC si utilizza l'unità di misura del *FLOPS* (FLoating-point Operations Per Second, "operazioni in virgola mobile al secondo").<br/>
Per valutare il vantaggio derivante dall'esecuzione di programmi paralleli in sistemi HPC si utilizzano alcune metriche: *speedup* ed *efficienza*.

Lo **Speedup** misura quanto è più veloce la versione parallela rispetto alla versione sequenziale (esprime il <ins>guadagno di un'applicazione parallela rispetto alla versione sequenziale</ins>). È pari a ```S = Tseq / Tpar```, dove *Tseq* è il tempo di esecuzione del programma nella versione sequenziale (su un solo nodo), e *Tpar* nella sua versione parallela.<br/>
Il caso ideale è che ```S = p```, dove *p* è il numero di processori. Tuttavia, solitamente ci sono altri fattori da considerare nell'equazione, quali ad esempio lo scarto di tempo dovuto all'*overhead*, pertanto in generale si ha che ```S < p```.

L'**Efficienza** misura lo <ins>speedup per numero di processori utilizzati</ins>. È pari a ```E = S / p```, dove *S* è lo speedup, e *p* il numero di processori utilizzati. Il caso ideale è che ```E = 1```, mentre nei casi reali si ha che ```E < 1```.

Un sistema si dice **scalabile** se mantiene la stessa efficienza al variare del numero di processori utilizzati e/o al variare della quantità di dati da elaborare.

La **Legge di Amdahl** considera che <ins>in generale non tutto il programma può essere parallelizzabile</ins>, dunque *Tpar* è dato da ```Tpar = r * Tseq + (1 - r) * Tseq / p```, dove *r* ∈ [0, 1] è una percentuale che esprime la frazione di tempo totale di esecuzione speso nella parte *non parallelizzabile* del programma. La Legge di Amdahl esprime lo speedup *S* come: ```S = Tseq / Tpar = 1 / (r + (1 - r) / p)``` e descrive l'andamento dello speedup al variare del numero di processori impiegati per la soluzione dello stesso problema. Se il numero dei processori tende a infinito, vediamo che la Legge di Amdahl ha un comportamento asintotico (per lim di *p* → ∞, *S* tende a 1/*r* senza mai toccarlo).

**Scalabilità Strong**: valuta l'<ins>efficienza al crescere del numero dei nodi</ins> (<ins>mantenendo costante la dimensione del problema</ins>). Lavoro totale da eseguire costante, ma lavoro da eseguire sul singolo nodo diminuisce al crescere del numero dei nodi (bilanciamento del lavoro sui nodi).

**Scalabilità Weak**: valuta l'<ins>efficienza al variare al crescere delle dimensioni del problema</ins> (<ins>mantenendo costante il carico di lavoro per singolo nodo</ins>). Per valutare la scalabilità weak si usano speedup scalato e efficienza scalata.

La **Legge di Gustafson** afferma che, <ins>assegnando ad ogni processore un workload costante</ins> ```(1 - r)```, lo <ins>speedup cresce linearmente con il numero dei processori</ins>, dunque lo speedup *S* è dato da: ```S = r + (1 - r) * p```.

### 4. Confrontare MPI ed OpenMP

Per ottenere i vantaggi del parallelismo, sfruttando efficacemente l'HW a disposizione, il programmatore deve trasformare i propri programmi seriali in codice parallelo. Per farlo è possibile utilizzare 2 approcci: <ins>parallelizzazione automatica</ins> (sfruttando ad esempio dei compilatori), che normalmente permettono di ottenere prestazioni non troppo soddisfacenti; <ins>parallelizzazione esplicita</ins>, utilizzando ad esempio dei linguaggi e librerie appositi per il calcolo parallelo.
A tal proposito esistono due modelli di interazione: scambio di messaggi (*MPI*) e memoria condivisa (*OpenMP*).

**MPI**: è uno <ins>standard</ins> che stabilisce un protocollo per la comunicazione fra processi in sistemi paralleli (*senza memoria condivisa*). Permette di eseguire più istanze di un programma in parallelo su più nodi.

**OpenMPI**: è una <ins>libreria</ins> per applicazioni parallele in sistemi a *memoria condivisa*.

<table>
	<tr>
		<td align="center" width="50%"><b>MPI</b></td>
		<td align="center" width="50%"><b>OpenMPI</b></td>
	</tr>
	<tr>
		<td align="center">Interazione basata su scambio di messaggi</td>
		<td align="center">Interazione basata su memoria condivisa</td>
	</tr>
	<tr>
		<td align="center">complessità d'uso</td>
		<td align="center">semplicità d'uso</td>
	</tr>
	<tr>
		<td align="center">load balancing a carico del programmatore</td>
		<td align="center">load balancing semplice da realizzare</td>
	</tr>
	<tr>
		<td align="center">elevata scalabilità</td>
		<td align="center">scalabilità limitata al numero di CPU disponibili sul nodo utilizzato</td>
	</tr>
	<tr>
		<td align="center">elevata portabilità (funziona anche su sistemi a memoria condivisa)</td>
		<td align="center">compatibile solo con sistemi a memoria condivisa (multicore/multiprocessors)</td>
	</tr>
</table>

