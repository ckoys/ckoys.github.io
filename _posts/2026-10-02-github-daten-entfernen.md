---
title: "Persönliche Daten wirklich aus einem GitHub-Repo entfernen"
date: 2026-10-02 12:00:00 +0200
tags:
  - Security
  - Git
---
Man muss es sich mal auf der Zunge zergehen lassen: Da arbeitet man beruflich in der IT-Security, predigt Datensparsamkeit, und auf dem eigenen, längst vergessenen Blog steht seit 2021 ein Impressum mit Wohnadresse und Handynummer. Öffentlich. Für jeden. Das Repo dahinter natürlich auch öffentlich, die Daten steckten also nicht nur in der Seite, sondern brav in jedem Commit seit dem ersten Tag.

Datei löschen, fertig? Ach bitte. Hier der Weg, der tatsächlich funktioniert, inklusive der zwei Stellen, an denen ich zielsicher falsch abgebogen bin.

## Warum `git rm` nichts bringt

Ein Commit, der die Datei löscht, entfernt sie aus dem aktuellen Stand. Schön. Jeder ältere Commit enthält sie trotzdem weiterhin, und jeder kann ihn auf GitHub aufrufen. Git vergisst nämlich nichts, das ist ja gerade der Witz an Git. Bei einem Jekyll-Blog kommt hinzu, dass oft auch der gebaute `_site/`-Ordner mit eingecheckt ist. Die Daten liegen dann zusätzlich als HTML im Feed, in der Sitemap und in der gerenderten Seite.

Bei mir standen sie an rund 70 Stellen der History, verteilt über Markdown, HTML und `feed.xml`. Gründlich war ich damals, das muss man mir lassen. Nur leider an der falschen Stelle.

## Schritt 1: aktuellen Stand bereinigen

Zuerst ein normaler Commit, der die Datei entfernt und alles aufräumt, was nicht ins Repo gehört (`_site/`, Caches, `.DS_Store`). So ist die Seite schon sauber, bevor die History umgeschrieben wird.

Vorher ein Backup:

```bash
git clone --mirror . ../repo-backup.git
```

## Schritt 2: History mit git-filter-repo umschreiben

[git-filter-repo](https://github.com/newren/git-filter-repo) ist das Werkzeug, das GitHub selbst empfiehlt. `filter-branch` und BFG darf man getrost in Rente schicken.

Zwei Dinge auf einmal: Pfade komplett entfernen und Textstellen in allen übrigen Dateien ersetzen. Die Ersetzungen kommen in eine Datei, eine Regel pro Zeile:

```text
Musterstraße 1==>[entfernt]
+49 151-00000000==>[entfernt]
12345 Musterstadt==>[entfernt]
```

```bash
git filter-repo --force \
  --invert-paths \
  --path _posts/2021-01-01-Impressum.md \
  --path _site/20210101/ \
  --replace-text ../replacements.txt
```

Wichtig: Die Regeln greifen auf jede Datei, auch auf Binärdateien. Deshalb vorher prüfen, in welchem Kontext ein Muster vorkommt (`git log --all -p | rg 'muster'`), und lieber lange, eindeutige Muster nehmen als eine nackte Postleitzahl.

`filter-repo` entfernt danach absichtlich das Remote `origin`, damit man nicht aus Versehen pusht. Sehr fürsorglich. Also neu eintragen und ganz bewusst force-pushen:

```bash
git remote add origin https://github.com/USER/REPO.git
git push --force -u origin main
```

Prüfen:

```bash
git log --all -p | rg -c 'Musterstraße|00000000' || echo sauber
```

## Schritt 3: GitHub selbst aufräumen lassen

Hier dachte ich, ich wäre fertig. Natürlich nicht. Der alte Commit war nach dem Force-Push weiterhin über seine SHA abrufbar:

```bash
gh api repos/USER/REPO/commits/<alte-sha>
```

Liefert fröhlich den alten Commit zurück. GitHub hält unerreichbare Commits und gecachte Ansichten vor, bis jemand sie aktiv entfernt. Und dieser Jemand ist nicht man selbst, sondern der Support.

**Falle Nummer eins:** Ich habe zuerst das Formular für „Private Information“ geöffnet. Klingt ja passend. Ist es aber nicht: Das ist für den Fall gedacht, dass *jemand anderes* deine Daten veröffentlicht hat. Sich selbst zu melden ist zwar eine originelle Idee, führt aber nirgendwohin.

Der richtige Weg steht in der GitHub-Doku [Removing sensitive data from a repository](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository), Abschnitt „Fully removing the data from GitHub“:

1. [support.github.com](https://support.github.com) öffnen, Konto wählen, Kategorie **Repositorys**.
2. Ticket mit diesen Angaben:
   - Owner und Repo-Name
   - Anzahl betroffener Pull Requests
   - die **First Changed Commits** aus `filter-repo`

**Falle Nummer zwei:** Die First Changed Commits, die der Support haben will, standen bei mir nicht in der Terminal-Ausgabe. Wo sind sie dann? In einer Datei, wo sonst:

```bash
cat .git/filter-repo/first-changed-commits
```

Und als Bonus ein Stolperstein im Formular selbst: Nach dem ersten „Weiter“ zeigt GitHub eine Seite mit Lösungsvorschlägen aus dem Community-Forum. Wer dort denkt, das Ticket sei raus, irrt (ich spreche aus Erfahrung). Erst „Weiter zur Ticketerstellung“, dann „Type of Issue“ wählen, dann „Submit“. Dann ist es raus. Wirklich.

## Schritt 4: was außerhalb von GitHub liegt

- **Forks:** Wer das Repo geforkt hat, hat die Daten. Bei mir: 0 Forks. Manchmal ist es eben ein Segen, dass sich niemand für den eigenen Blog interessiert.
- **Wayback Machine:** Über `https://web.archive.org/cdx/search/cdx?url=domain.tld*` sieht man, welche Seiten archiviert sind. Bei mir nur Snapshots von 2020, das Impressum war nie drin.
- **Lokale Kopien:** Das Backup aus Schritt 1 und die Datei mit den Ersetzungsregeln enthalten die Daten im Klartext. Beides löschen, sobald alles verifiziert ist.

## Was ich daraus mitnehme

Persönliche Daten gehören nicht in ein Git-Repo. Auch nicht in ein „privates Spielprojekt“, das ganz zufällig öffentlich ist. Wer ein Impressum braucht, verlinkt besser auf eine zentrale Seite, die man an genau einer Stelle pflegt.

Und der Force-Push ist nicht das Ende der Bereinigung, sondern ungefähr die Mitte. Wer das für übertrieben hält, darf gern mal `gh api` auf die eigene alte SHA loslassen. Viel Spaß.

