# Terraform: Moduler, Remote State og CI/CD

I denne øvelsen skal du bygge en gjenbrukbar Terraform-modul som hoster en statisk nettside på AWS. Du lærer å organisere Terraform-kode i moduler, dele state i S3, legge CloudFront foran bucketen og automatisere deployment med GitHub Actions.

## Du vil lære

- **Terraform-moduler**: Pakke infrastruktur i gjenbrukbare komponenter
- **Remote state**: Dele Terraform state i S3 og låse den for samtidige endringer
- **CloudFront CDN**: Global distribusjon med HTTPS foran S3
- **CI/CD med GitHub Actions**: Automatisere `terraform plan` og `apply`

## AWS-tjenester i denne labben

- **S3 (Simple Storage Service)**: Objektlager. Hoster de statiske filene som utgjør nettsiden, og lagrer Terraform state remote.
- **CloudFront**: AWS sitt CDN. Distribuerer nettsiden globalt og legger HTTPS på toppen av S3.
- **IAM**: Identity and Access Management. Styrer gjennom bucket policies og GitHub Actions-credentials hvem som kan lese og endre hva.

## Forberedelser

### Om GitHub forks

En **fork** er din egen kopi av et GitHub-repo under din GitHub-konto. Du jobber i din kopi uten å påvirke originalen, og kan senere åpne pull requests tilbake hvis du vil bidra endringer. I denne labben trenger du en fork for å kunne pushe commits (Del 4 krever det) og legge inn repository secrets i ditt eget repo.

### Steg 0: Opprett GitHub Codespace fra din fork

1. **Fork dette repositoriet** til din egen GitHub-konto
2. **Åpne Codespace**: Klikk på "Code" → "Codespaces" → "Create codespace on main"
3. **Vent på at Codespace starter**: Dette kan ta et par minutter første gang

### Konfigurer AWS-nøkler i Codespace

Terraform og AWS CLI trenger AWS-nøkler for å kunne snakke med AWS-kontoen din. En Codespace starter uten disse.

Hent `Access Key ID` og `Secret Access Key` fra AWS Academy / IAM, og kjør:

```bash
aws configure
```

Fyll inn verdiene:

- **AWS Access Key ID**: fra kontoen din
- **AWS Secret Access Key**: fra kontoen din
- **Default region name**: `eu-west-1`
- **Default output format**: `json`

Hvis du bruker AWS Academy må du i tillegg sette `AWS_SESSION_TOKEN`. Verifiser at nøklene fungerer:

```bash
aws sts get-caller-identity
```

Kommandoen skal returnere konto-ID og bruker-ARN.

---

## Del 1: Remote State Management

### Hvorfor Remote State?

Når flere personer jobber med samme infrastruktur, eller når vi skal automatisere med CI/CD, trenger vi en felles plass å lagre Terraform state. Lokal state fungerer ikke i team-miljøer.

### Steg 1: Opprett `providers.tf`

Lag `providers.tf` i rotmappen:

```hcl
terraform {
  required_version = ">= 1.10"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-west-1"
}
```

### Steg 2: Konfigurer Backend

Klassen har en felles S3 bucket for Terraform state: `pgr301-terraform-state` i `eu-west-1`. Du trenger ikke opprette din egen — bruk denne, men gi din state en unik `key` slik at du ikke overskriver medstudenter.

Opprett `backend.tf` i rotmappen:

```hcl
terraform {
  backend "s3" {
    bucket       = "pgr301-terraform-state"
    key          = "ola-nordmann/website/terraform.tfstate"  # Bytt "ola-nordmann" til ditt eget navn
    region       = "eu-west-1"
    use_lockfile = true
    encrypt      = true
  }
}
```

`key` er stien til din state-fil inne i bucketen. Prefikset (f.eks. `ola-nordmann/`) skiller din state fra andre studenters. `use_lockfile = true` ber Terraform låse state via en lås-fil i selve S3-bucketen (støttet fra Terraform 1.10).

### Steg 3: Initialiser Terraform

```bash
terraform init
```

Dette validerer backend-konfigurasjonen og laster ned AWS-provideren. State-filen i S3 blir først opprettet når du kjører `terraform apply` i Del 2.

## Del 2: Terraform-moduler - Gjenbrukbar Infrastruktur

### Hva er moduler?

Moduler er Terraforms måte å pakke og gjenbruke infrastruktur-kode på. I stedet for å duplisere kode, lager vi en modul som kan brukes flere steder med ulike konfigurasjoner. Strukturelt ligner en modul på en funksjon i et programmeringsspråk: variablene er input, ressursene er logikken, og outputs er returverdiene.

### Modulstruktur

En typisk Terraform-modul består av følgende filer:

```
modules/s3-website/
├── main.tf        # Hovedressurser (S3, CloudFront, etc.)
├── variables.tf   # Input-variabler som modulen tar imot
└── outputs.tf     # Output-verdier som modulen returnerer
```

**Fordeler med moduler:**
- **Gjenbrukbarhet**: Samme modul kan brukes i flere prosjekter eller miljøer
- **Abstrahering**: Skjuler kompleksitet bak et enkelt grensesnitt
- **Standardisering**: Sikrer konsistent infrastruktur på tvers av prosjekter
- **Vedlikehold**: Endringer på ett sted propagerer til alle bruksområder

### Steg 1: Opprett modul-struktur

Lag mappestrukturen for modulen:

```bash
mkdir -p modules/s3-website
touch modules/s3-website/main.tf
touch modules/s3-website/variables.tf
touch modules/s3-website/outputs.tf
```

- `mkdir -p` oppretter mappen og eventuelle manglende mellomliggende mapper
- `touch` oppretter tomme filer som fylles inn i de neste stegene

### Steg 2: Definer variabler for modulen

Variabler gjør modulen gjenbrukbar — samme modul kan brukes med ulike verdier for forskjellige miljøer.

**Fyll inn** `modules/s3-website/variables.tf`:

```hcl
variable "bucket_name" {
  description = "Name of the S3 bucket"
  type        = string
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default     = {}
}
```

### Steg 3: Lag ressursene i modulen

**Fyll inn** `modules/s3-website/main.tf` med S3-ressursene for en statisk nettside:

```hcl
resource "aws_s3_bucket" "website" {
  bucket = var.bucket_name
  tags   = var.tags
}

resource "aws_s3_bucket_website_configuration" "website" {
  bucket = aws_s3_bucket.website.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}

resource "aws_s3_bucket_public_access_block" "website" {
  bucket = aws_s3_bucket.website.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_policy" "website" {
  bucket = aws_s3_bucket.website.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "PublicReadGetObject"
        Effect    = "Allow"
        Principal = "*"
        Action    = "s3:GetObject"
        Resource  = "${aws_s3_bucket.website.arn}/*"
      }
    ]
  })

  depends_on = [aws_s3_bucket_public_access_block.website]
}
```

### Steg 4: Definer outputs for modulen

**Fyll inn** `modules/s3-website/outputs.tf`:

```hcl
data "aws_region" "current" {}

output "bucket_name" {
  description = "Name of the S3 bucket"
  value       = aws_s3_bucket.website.id
}

output "website_url" {
  description = "URL of the S3 website"
  value       = "http://${aws_s3_bucket.website.bucket}.s3-website.${data.aws_region.current.name}.amazonaws.com"
}

output "bucket_arn" {
  description = "ARN of the S3 bucket"
  value       = aws_s3_bucket.website.arn
}
```

### Steg 5: Bruk modulen fra rot-prosjektet

Opprett `main.tf` i rotmappen (ikke inne i modulen) som kaller modulen. Bytt ut bucket_name med et unikt bucketnavn!

```hcl
module "s3_website" {
  source = "./modules/s3-website"

  bucket_name = "ola-nordmann-pgr301-website"  # Bytt til noe globalt unikt (f.eks. ditt-navn-pgr301-website)

  tags = {
    Name        = "PGR301 Lab"
    Environment = "Demo"
    ManagedBy   = "Terraform"
  }
}

output "s3_website_url" {
  value       = module.s3_website.website_url
  description = "URL for the S3 hosted website"
}

output "bucket_name" {
  value       = module.s3_website.bucket_name
  description = "Name of the S3 bucket"
}
```

**Viktig om bucket-navn**: S3 bucket-navn må være **globalt unike** på tvers av alle AWS-kontoer. Bruk noe som dine initialer eller studentnummer kombinert med `pgr301-website`.

### Steg 6: Deploy

```bash
terraform init    # Re-init siden du har lagt til en modul
terraform plan
terraform apply
```

Når apply er ferdig, skal du se `s3_website_url` som output. Verifiser også at state-filen nå ligger i S3-bucketen `pgr301-terraform-state` under din `key`.

### Steg 7: Last opp nettsiden

Repoet inneholder en enkel statisk nettside i `website/`-mappen. Last opp filene til din bucket:

```bash
aws s3 sync website/ s3://ola-nordmann-pgr301-website
```

- `aws s3 sync` kopierer filer og speiler kataloginnhold
- Bytt bucket-navnet til ditt eget

Hent URL-en og åpne den i nettleseren:

```bash
terraform output s3_website_url
```

### Utfordring (ekstra)

Legg til en `enable_versioning`-variabel i modulen som gjør versioning valgfri:

```hcl
resource "aws_s3_bucket_versioning" "website" {
  count  = var.enable_versioning ? 1 : 0
  bucket = aws_s3_bucket.website.id
  # ...
}
```

Hint: Bruk `count` eller `for_each` basert på variabelen.

---

## Del 3: CloudFront CDN

### Hvorfor CloudFront?

S3 website hosting har begrensninger:
- Ingen HTTPS-støtte
- Ikke globalt distribuert (treg for brukere langt fra bucket-regionen)

CloudFront løser disse problemene.

### Legg til CloudFront Distribution

**Utvid** `modules/s3-website/main.tf` med CloudFront:

```hcl
resource "aws_cloudfront_distribution" "website" {
  enabled             = true
  default_root_object = "index.html"

  origin {
    domain_name = aws_s3_bucket_website_configuration.website.website_endpoint
    origin_id   = "S3-${var.bucket_name}"

    custom_origin_config {
      origin_protocol_policy = "http-only"
      http_port              = 80
      https_port             = 443
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  default_cache_behavior {
    target_origin_id       = "S3-${var.bucket_name}"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 0
    default_ttl = 0  # Instant refresh - ingen caching
    max_ttl     = 0
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
```

### Legg til CloudFront Output

**Utvid** `modules/s3-website/outputs.tf`:

```hcl
output "cloudfront_url" {
  description = "CloudFront distribution URL (HTTPS enabled)"
  value       = "https://${aws_cloudfront_distribution.website.domain_name}"
}

output "cloudfront_domain" {
  description = "CloudFront domain name"
  value       = aws_cloudfront_distribution.website.domain_name
}
```

### Oppdater Rot-Outputs

I rot-`main.tf`, legg til CloudFront output:

```hcl
output "cloudfront_url" {
  value       = module.s3_website.cloudfront_url
  description = "CloudFront URL with HTTPS"
}
```

### Deploy CloudFront

```bash
terraform apply
```

CloudFront-deployment tar typisk 3-15 minutter.

### Test CDN

```bash
terraform output cloudfront_url
```

Åpne URL-en i nettleseren. HTTPS fungerer automatisk, og URL-en er global (CloudFront, ikke region-spesifikk).

---

## Del 4: GitHub Actions CI/CD

### Mål

Automatiser Terraform deployment:
- **Pull Request**: Kjør `terraform plan` og vis endringer i PR
- **Merge til main**: Kjør `terraform apply` automatisk

### Steg 1: Opprett Workflow-fil

Lag `.github/workflows/terraform.yml`:

```yaml
name: Terraform Infrastructure

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  AWS_REGION: eu-west-1
  TF_VERSION: 1.10.0

jobs:
  terraform:
    name: Terraform Plan & Apply
    runs-on: ubuntu-latest

    permissions:
      pull-requests: write
      contents: read

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format Check
        run: terraform fmt -check

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color  # -no-color gir pen output i PR-kommentar
        continue-on-error: true

      - name: Comment Plan on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const output = `### Terraform Plan

            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`

            *Pushed by: @${{ github.actor }}*`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve  # -auto-approve hopper over interaktiv bekreftelse
```

GitHub Actions ser kun workflows som ligger i repoet på GitHub. Commit og push workflow-filen før du går videre — ellers trigges ingenting:

```bash
git add .github/workflows/terraform.yml
git commit -m "Add Terraform CI/CD workflow"
git push
```

### Steg 2: Konfigurer GitHub Secrets

Gi GitHub Actions tilgang til AWS:

1. Gå til ditt GitHub repository
2. **Settings** → **Secrets and variables** → **Actions**
3. Klikk **New repository secret** og legg til to secrets med dine egne verdier:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`

Disse secrets bør være fra en dedicated IAM-bruker med minimal permissions (kun det Terraform trenger).

### Steg 3: Test Pipeline

1. Lag en ny branch:

```bash
git checkout -b test-pipeline
```

2. Gjør en liten endring i rot-`main.tf` — legg til `PipelineTest = "true"` i tags-blokken:

```hcl
module "s3_website" {
  # ...
  tags = {
    # ...
    PipelineTest = "true"
  }
}
```

Workflowen kjører `terraform fmt -check`. Kjør `terraform fmt` lokalt først — den formatterer koden korrekt så pipelinen ikke feiler:

```bash
terraform fmt
```

3. Commit og push:

```bash
git add .
git commit -m "Test GitHub Actions pipeline"
git push origin test-pipeline
```

4. Opprett Pull Request på GitHub
5. GitHub Actions kjører `terraform plan`, og plan-outputen vises som kommentar på PR-en
6. Merge PR til main — da kjører `terraform apply` automatisk

---

## Oppsummering

Du har nå lært:

- **Remote State Management**: State-deling i team og CI/CD
- **Terraform-moduler**: Gjenbrukbar infrastruktur-kode
- **CloudFront CDN**: Global distribusjon med HTTPS
- **CI/CD med GitHub Actions**: Automatisert `terraform plan` og `apply`

---

## Opprydding

Når du er ferdig med labben, rydd opp etter deg for å unngå løpende AWS-kostnader og søppel i den delte state-bucketen.

1. Tøm nettside-bucketen og destroy infrastrukturen:

```bash
aws s3 rm s3://ola-nordmann-pgr301-website --recursive  # Bytt til ditt bucket-navn
terraform destroy
```

2. Slett din state-fil fra den delte bucketen:

```bash
aws s3 rm s3://pgr301-terraform-state/ola-nordmann/website/terraform.tfstate  # Bytt prefikset til ditt eget
```

---

## Bonusoppgave: Validation Rules

Legg til validation i `modules/s3-website/variables.tf` slik at modulen avviser ugyldige bucket-navn før `terraform apply`:

```hcl
variable "bucket_name" {
  description = "Name of the S3 bucket"
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]*[a-z0-9]$", var.bucket_name))
    error_message = "Bucket name must start and end with lowercase letter or number, and contain only lowercase letters, numbers, and hyphens."
  }
}
```

Test ved å prøve en ugyldig verdi (f.eks. `Bucket_Med_Store_Bokstaver`) og se at `terraform plan` feiler med din feilmelding.

---

## Ressurser

- [Terraform Modules Documentation](https://developer.hashicorp.com/terraform/language/modules)
- [AWS CloudFront Developer Guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/)
- [GitHub Actions Terraform Tutorial](https://developer.hashicorp.com/terraform/tutorials/automation/github-actions)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)
