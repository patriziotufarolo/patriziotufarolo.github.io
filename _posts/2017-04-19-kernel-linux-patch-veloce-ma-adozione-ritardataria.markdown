---
layout: post
title: Kernel Linux, patch veloce ma adozione ritardataria
date: '2017-04-19 07:49:04'
tags:
- sicurezza
- tecnologia
- bug
- kernel-linux
- android
- cve-2016-10229
---

Via [Punto-Informatico.it](http://punto-informatico.it) - [Link all'articolo](http://punto-informatico.it/4382623/PI/News/kernel-linux-patch-veloce-ma-adozione-ritardataria.aspx)

*Le maggiori distro GNU/Linux hanno da poco introdotto una patch rilasciata più di un anno fa per un pericoloso bug che consentirebbe l'esecuzione di codice arbitrario via UDP. Ma il rischio per i dispositivi Android ed embedded resta alto*

*Roma* - Già nota dagli ultimi mesi del 2015, cui è seguita una patch silente ma immediata, la vulnerabilità [CVE-2016-10229](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-10229) di tipo *arbitrary code execution*, che riguarda le versioni del kernel Linux comprese tra la 2.6 e la 4.5, è stata catalogata nel *Common Vulnerability Scoring System v3* con un **punteggio pari a 9.8**, ma la patch è stata inclusa solo di recente nelle maggiori distribuzioni GNU/Linux.

Il problema deriva dalla **gestione del flag MSG_PEEK nella funzione recvmsg relativa al protocollo UDP**, il cui ruolo è quello di permettere il prefetch di un messaggio in arrivo su un socket UDP senza consumare il buffer di lettura.
In particolare, quando l'utente fornisce una quantità di dati inferiore alla dimensione del socket buffer (*SKB*), il checksum del pacchetto UDP viene calcolato due volte; la seconda volta utilizzando una funzione insicura (*skb\_copy\_and\_csum\_datagram\_iovec*) che potrebbe causare l'esecuzione di codice arbitrario.

Questa funzione, nei kernel del ramo mainline, [è stata già sostituita](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=197c949e7798fbf28cfadc69d9ca0c2abbf93191) da tempo con l'equivalente *skb\_copy\_and\_csum\_datagram\_msg*, in grado di gestire i buffer corti in modo sicuro, risolvendo parzialmente il problema.L'autore della patch (poche e semplici righe di codice) è Eric Dumazet (Google), che [minimizza l'entità della vulnerabilità](https://plus.google.com/+EricDumazet/posts/ZQie5XjAic2) sottolineando come i kernel non patchati con il commit [89c22d8c3b27 ("net: Fix skb csum races when peeking")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=89c22d8c3b27) ne siano esenti. 
È il caso ad esempio dei [kernel Red Hat](https://access.redhat.com/security/cve/cve-2016-10229); al contrario, i manutentori di altre distribuzioni come Ubuntu e Debian hanno distribuito una versione sicura del kernel nello scorso mese di febbraio.

Sebbene i kernel messi in sicurezza siano stati rilasciati ormai da tempo, **l'impatto della vulnerabilità non è da sottovalutare**. Bisogna considerare che il flag *MSG_PEEK* è molto utilizzato sia in software noti che in librerie comunemente adottate dagli sviluppatori, come riscontrabile da ricerche su [Github](https://github.com/search?l=C&q=MSG_PEEK&type=Code) o su [SearchCode](https://searchcode.com/?q=MSG_PEEK).
È quindi raccomandabile usare una versione aggiornata del kernel, anche nell'ottica di contrastare molte [altre vulnerabilità recentemente scoperte](http://www.cvedetails.com/vulnerability-list.php?vendor_id=33&product_id=47&version_id=&page=1&hasexp=0&opdos=0&opec=0&opov=0&opcsrf=0&opgpriv=0&opsqli=0&opxss=0&opdirt=0&opmemc=0&ophttprs=0&opbyp=0&opfileinc=0&opginf=0&cvssscoremin=8&cvssscoremax=0&year=2017&month=0&cweid=0&order=1&trc=247&sha=d80d3346f69d7155f090a2b7862af859427c62ef).

Una riflessione particolare va fatta nei confronti dei **dispositivi mobile Android**, destinati a rimanere ancora a lungo vulnerabili a causa del modello di erogazione degli aggiornamenti del sistema operativo di casa Google. Il 5 Aprile il team Android [ha rilasciato un aggiornamento](https://source.android.com/security/bulletin/2017-04-01) contenente, tra le altre, la soluzione per il bug esposto. Tuttavia, la distribuzione degli aggiornamenti è in carico ai principali produttori mobile: [Samsung](http://security.samsungmobile.com/smrupdate.html#SMR-APR-2017) e [LG](https://lgsecurity.lge.com/security_updates.html) hanno già risposto alla chiamata da parte di Mountain View con un aggiornamento di sicurezza; sarà quindi necessario valutare quanti e quali saranno effettivamente i dispositivi coinvolti in mano agli utenti.
Inoltre, la patch è stata applicata al [kernel di Lineage OS](https://github.com/LineageOS/android_kernel_google_msm/commit/8711b467e96e1172c595ca7b3eaf15918282ce6a), da cui derivano molte *custom rom*.

Infine, trattandosi di vulnerabilità del kernel Linux, è inevitabile non menzionare l'impatto delle stesse sui dispositivi embedded (ad esempio apparati di rete domestici come router e device IoT), per i quali l'onere dell'aggiornamento è in capo ai proprietari, spesso utenti ignari del livello di rischio cui si è esposti sulla rete Internet.

Patrizio Tufarolo

<img src="/content/images/2017/04/cc.logo_.large_.png" title="Creative Commons" width="200" style="margin:auto; display:block;">