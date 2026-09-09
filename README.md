# Portale ITS

Generatore statico del catalogo corsi ITS ICT Piemonte. Il sito viene costruito
da `data/corsi.json` con Node.js e pubblicato come file statici nella cartella
`dist/`.

## Prerequisiti

- Node.js con supporto ai moduli ECMAScript
- npm
- Terraform >= 1.6, solo per validare o gestire l'infrastruttura AWS
- Python, opzionale, per avviare un server HTTP locale

Il progetto applicativo non ha dipendenze npm esterne.

## Flusso locale

1. Modifica `data/corsi.json`.
2. Esegui i test:

	```powershell
	npm test
	```

	I test verificano che il catalogo non sia vuoto, che ogni corso abbia un
	codice valido, un titolo e un numero di ore positivo, che i totali siano
	corretti e che l'HTML generato non contenga pattern di segreti.

3. Genera il sito:

	```powershell
	npm run build
	```

	La build crea:

	- `dist/index.html`, la pagina del catalogo;
	- `dist/style.css`, copiato da `site/style.css`;
	- `dist/versione.json`, con versione, numero di corsi e totale ore.

4. Controlla il risultato generato:

	```powershell
	Get-Content .\dist\versione.json
	Start-Process (Resolve-Path .\dist\index.html)
	```

	Per provarlo tramite HTTP invece di aprire direttamente il file:

	```powershell
	python -m http.server 8000 --directory dist
	```

	Apri quindi `http://localhost:8000` nel browser.

## Controllo anti-segreti

Su Windows, esegui il controllo equivalente con PowerShell:

```powershell
$patterns = 'AKIA[0-9A-Z]{16}','ghp_[A-Za-z0-9]{36}','-----BEGIN [A-Z ]*PRIVATE KEY','AWS_SECRET_ACCESS_KEY\s*[:=]\s*[A-Za-z0-9/+]{40}','(password|passwd|api[_-]?token)\s*[:=]\s*.{8,}'
$matches = Get-ChildItem -Recurse -File |
  Where-Object { $_.FullName -notmatch '\\(\.git|node_modules|\.terraform|dist)\\' -and $_.Name -notin 'package-lock.json','sniffa-segreti.sh' } |
  Select-String -Pattern $patterns
if ($matches) { $matches; exit 1 } else { 'nessun segreto sospetto trovato' }
```

Lo script `script/sniffa-segreti.sh` contiene lo stesso controllo per Bash.
Poiche il file e salvato con terminatori Windows (`CRLF`), in Bash potrebbe
essere necessario convertirlo prima in formato Unix:

```bash
sed -i 's/\r$//' script/sniffa-segreti.sh
bash script/sniffa-segreti.sh .
```

## Validazione Terraform

La configurazione Terraform si trova in `infra/tf/` e definisce il bucket del
sito, il bucket dei log e la tabella DynamoDB delle iscrizioni.

Controlla la formattazione:

```powershell
terraform -chdir=infra/tf fmt -check
```

Inizializza i provider senza configurare un backend remoto e valida la sintassi:

```powershell
terraform -chdir=infra/tf init -backend=false
terraform -chdir=infra/tf validate
```

Per vedere le modifiche che verrebbero applicate su AWS, usa un account e
credenziali configurati correttamente e lancia prima:

```powershell
terraform -chdir=infra/tf plan -var="env=test" -var="owner=squadra-0"
```

Non eseguire `apply` senza avere prima esaminato il piano e verificato account,
regione e variabili. `aws_endpoint` permette di puntare a un endpoint AWS
compatibile locale, come Moto, durante i test di infrastruttura.

## Verifica completa consigliata

Dopo ogni modifica ai dati o al codice, esegui in questo ordine:

```powershell
npm test
npm run build
terraform -chdir=infra/tf fmt -check
terraform -chdir=infra/tf validate
```

Il deploy manuale descritto in `DEPLOY.md` e obsoleto: non usare credenziali
memorizzate in file locali e non rendere pubblico il bucket per risolvere un
errore di permessi senza aver prima verificato la configurazione AWS.