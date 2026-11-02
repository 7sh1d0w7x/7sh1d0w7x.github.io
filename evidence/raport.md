# A9 - backup si restaurare cronometrata (2 nov 2026)

**Host:** ulab (Ubuntu 26.04, VM KVM) · **Unealta:** restic 0.18 · **Rulat manual de owner**

---

## Ce s-a facut

Ciclu complet, cu dezastru simulat intre backup si restaurare:

```
   1. date de test .......... 80 MB, 21 fisiere (o "baza de date" de 60 MB + 20 loguri)
   2. manifest inainte ...... sha256 pe toate fisierele, salvat in afara directorului
   3. repo restic ........... /srv/backup (init, parola proprie)
   4. backup ................ primul, complet; apoi inca unul fara modificari
   5. dezastru .............. rm -rf /srv/appdata
   6. restaurare ............ restic restore --target /, la caile originale
   7. verificare ............ hash-uri comparate cu manifestul
   8. automatizare .......... systemd service + timer orar
```

## Cifrele masurate

| Faza | Timp | Observatii |
|---|---|---|
| Primul backup | **12,8 s** | 80.012 MiB scrisi in repo, fara compresie (date aleatorii) |
| Al doilea backup | **8,6 s** | 0 B adaugati, doar citire si hash |
| Restaurare | **5,1 s** | 25 intrari (21 fisiere + 4 directoare) |
| Repo dupa 2 snapshot-uri | **82 MB** | 80 MB date + index si metadate |

**Integritate:** 21 din 21 hash-uri identice dupa restaurare.

## Ce a mers prost, si ce s-a invatat

**1. Comparatia initiala a esuat, desi restaurerea era corecta.** `diff` a raportat 21 de diferente pentru ca manifestul vechi avea cai absolute, iar cel nou relative. Hash-urile erau identice. Lectia: se compara hash-urile, nu liniile intregi.

**2. Serviciul systemd a picat la prima rulare.** Motivul, din jurnal: `Fatal: /etc/restic/password does not exist`. Pasul de creare a fisierului cu parola fusese sarit. Lectia: eroarea spune exact ce lipseste, nu trebuie ghicit.

**3. Retentia nu a sters nimic la prima incercare.** `--keep-daily 7` pastreaza cel mai vechi snapshot plus ultimul din fiecare zi, iar ambele snapshot-uri erau din aceeasi zi. `--keep-last 1` a sters corect. Lectia: politicile pe zile nu taie nimic daca toate snapshot-urile sunt de azi.

**4. Parola de repo a fost gresita de trei ori la taste.** Nu e ghinion, e frecare. Rezolvarea: `--password-file` cu permisiuni 600, ceea ce serviciul foloseste deja.

## Automatizare

```
   /etc/restic/password ................ 600, citit cu --password-file
   restic-backup.service ............... oneshot; backup, apoi forget --prune
   restic-backup.timer ................. orar, Persistent=true, RandomizedDelaySec=300
   prima rulare programata ............. 20:03 UTC (20:00 + intarziere aleatoare)
```

`ExecStartPost` ruleaza doar daca backup-ul a reusit, deci o eroare de backup nu declanseaza curatenia.

## Ce NU s-a dovedit

```
   · backup offsite real: repo-ul e pe acelasi host (/srv/backup), nu pe alta masina
   · dezastru fizic: disc cazut, pană de curent, ransomware
   · volum real: 80 MB nu spune nimic despre 50 GB
   · restaurare partiala: doar "tot" s-a testat, nu "un singur fisier din 5000"
   · test de restaurare programat: backup-ul e automat, restaurarea inca e manuala
```

## Ce ar transforma asta in livrabil de client

```
   1. tinta offsite (sftp sau s3), nu acelasi disc
   2. un test de restaurare lunar, automat, cu raport (nu doar backup)
   3. timp masurat pe volumul real al clientului, nu pe 80 MB
   4. document scris: ce se salveaza, unde, cat de des, cat dureaza refacerea
```

Astea trei sunt exact ce separa "am backup" de "pot dovedi ca refac serverul".
