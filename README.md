# Terraform: Moduler, Remote State og CI/CD

Denne oppgaven bygger videre på [terraform-s3-website](https://github.com/glennbech/terraform-s3-website). Du skal utvide infrastrukturen med mer avanserte Terraform-konsepter:

- Remote state management for team-samarbeid
- Terraform-moduler for gjenbrukbar infrastruktur
- CloudFront CDN for global distribusjon
- Automatisering med GitHub Actions

**Forutsetninger**: Du har fullført del 1 (terraform-s3-website) og har en fungerende S3-nettside deployet via Terraform.

---

## Del 1: Remote State Management

### Hvorfor Remote State?

Når flere personer jobber med samme infrastruktur, eller når vi skal automatisere med CI/CD, trenger vi en felles plass å lagre Terraform state. Lokal state fungerer ikke i team-miljøer.

### Steg 1: Konfigurer Backend

Klassen har en felles S3 bucket for Terraform state: `pgr301-terraform-state` i `eu-west-1`. Du trenger ikke opprette en egen bucket — bruk denne, men gi din state en unik `key` slik at du ikke overskriver medstudenter.

1. **Opprett fil** `backend.tf` i rotmappen:

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

`key` er stien til din state-fil inne i bucketen. Prefikset (f.eks. `ola-nordmann/`) skiller din state fra andre studenters. `use_lockfile = true` ber Terraform om å låse state via en lås-fil i selve S3-bucketen (støttet fra Terraform 1.10).

2. **Migrer state til remote backend**:

```bash
terraform init -migrate-state
```

Terraform vil spørre om du vil kopiere eksisterende state til det nye backend. Svar `yes`.

3. **Verifiser**:
   - Gå til S3 Console og se at state-filen er lastet opp
   - Din lokale `terraform.tfstate` skal nå være tom eller borte

State er nå lagret sentralt. Hvis flere personer jobber på samme prosjekt, vil de alle dele samme state. I tillegg er dere nå beskyttet mot at flere gjør `terraform apply` samtidig — S3-låsefilen sørger for state locking slik at kun én person kan gjøre endringer om gangen.

### Test State Locking

Du kan teste state locking ved å åpne to terminaler og prøve å kjøre `terraform apply` i begge samtidig:

1. **Terminal 1**: Kjør `terraform apply` og bekreft med `yes`
2. **Terminal 2**: Kjør raskt `terraform apply` mens Terminal 1 fortsatt jobber

Vær rask: `terraform apply` fullføres raskt når det ikke er mange endringer.

Du vil se at Terminal 2 får en feilmelding om at state er låst, med informasjon om hvem som holder låsen. Dette forhindrer at to personer gjør motstridende endringer samtidig.

---

## Del 2: Terraform-moduler - Gjenbrukbar Infrastruktur

### Hva er moduler?

Moduler er Terraforms måte å pakke og gjenbruke infrastruktur-kode på. I stedet for å duplisere kode, lager vi en modul som kan brukes flere steder med ulike konfigurasjoner. Strukturelt ligner en modul på en funksjon i et programmeringsspråk: variablene er input, ressursene er logikken, og outputs er returverdiene.

### Modulstruktur

En typisk Terraform-modul består av følgende filer:

```
modules/s3-website/
├── main.tf        # Hovedressurser (S3, CloudFront, etc.)
├── variables.tf   # Input-variabler som modulen tar imot
├── outputs.tf     # Output-verdier som modulen returnerer
└── versions.tf    # Provider requirements (valgfri)
```

**Fordeler med moduler:**
- **Gjenbrukbarhet**: Samme modul kan brukes i flere prosjekter eller miljøer
- **Abstrahering**: Skjuler kompleksitet bak et enkelt grensesnitt
- **Standardisering**: Sikrer konsistent infrastruktur på tvers av prosjekter
- **Vedlikehold**: Endringer på ett sted propagerer til alle bruksområder

### Provider Configuration i Moduler

I denne oppgaven bruker vi **to AWS providers** (multi-region setup):
- Default provider i `eu-west-1` for S3, CloudFront, etc.
- Aliased provider i `us-east-1` for ACM-sertifikater (CloudFront-krav)

**Viktig**: Du må sende begge providers til modulen eksplisitt. Se [Appendix A: Provider Configuration](#appendix-a-provider-configuration-i-moduler) for detaljert forklaring av hvordan dette fungerer.

**I rot-nivå `main.tf`, send providers til modulen**:

```hcl
module "s3_website" {
  source = "./modules/s3-website"

  providers = {
    aws           = aws           # Default provider
    aws.us-east-1 = aws.us-east-1 # Aliased provider
  }

  bucket_name = var.bucket_name
  subdomain   = var.subdomain
}
```

### Del A: Lag en modul

#### Steg 1: Opprett modul-struktur

Lag mappestrukturen for modulen slik:

```bash
mkdir -p modules/s3-website
touch modules/s3-website/main.tf
touch modules/s3-website/variables.tf
touch modules/s3-website/outputs.tf
```

**Forklaring:**
- `mkdir -p` oppretter mappen (og eventuelle manglende mellomliggende mapper); `-p` gjør at kommandoen ikke feiler hvis mappen allerede finnes
- `touch` oppretter en tom fil hvis den ikke finnes fra før (ellers oppdaterer den bare timestampet) — vi lager tre tomme `.tf`-filer som fylles inn i de neste stegene

Dette gir følgende struktur:

```
modules/
└── s3-website/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

#### Steg 2: Definer variabler for modulen

Variabler er det som gjør moduler gjenbrukbare - de lar deg bruke samme modul med ulike verdier for forskjellige miljøer eller brukstilfeller. Uten variabler ville modulen alltid opprette de samme ressursene med de samme verdiene, noe som ville gjøre den ubrukelig for gjenbruk.

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

#### Steg 3: Flytt Ressurser til modulen

**Flytt S3-ressursene** fra root `main.tf` til `modules/s3-website/main.tf`:

- Kopier alle S3-relaterte ressurser (`aws_s3_bucket`, `aws_s3_bucket_website_configuration`, etc.)
- Erstatt hardkodede verdier med `var.bucket_name`, `var.tags`, etc.

**Hint**: I modulen skal du bruke `var.bucket_name` direkte.

#### Steg 4: Definer outputs for modulen

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

### Før du bruker modulen: Destroy først

Tøm bucketen og destroy eksisterende infrastruktur før du tar modulen i bruk:

```bash
aws s3 rm s3://ditt-bucket-navn --recursive
terraform destroy
```

### Del B: Bruk modulen

Nå skal du refaktorere root `main.tf` til å bruke modulen du nettopp laget.

#### Din oppgave:

1. **I root `main.tf`**: Erstatt alle S3-ressursene med et modul-kall:

```hcl
module "s3_website" {
  source = "./modules/s3-website"

  bucket_name         = "ditt-bucket-navn"

  tags = {
    Name        = "Crypto Juice Exchange"
    Environment = "Demo"
    ManagedBy   = "Terraform"
  }
}
```

2. **Oppdater outputs** i root `main.tf` til å bruke module outputs:

```hcl
output "s3_website_url" {
  value       = module.s3_website.website_url
  description = "URL for the S3 hosted website"
}

output "bucket_name" {
  value       = module.s3_website.bucket_name
  description = "Name of the S3 bucket"
}
```

3. **Test konfigurasjonen**:

```bash
terraform init  # Re-initialiser for modulen
terraform plan
terraform apply
```

**Forventet resultat**: Terraform skal si at det ikke er noen endringer nødvendig (hvis du har flyttet alt riktig).

#### Utfordring (ekstra):

- Kan du legge til en `enable_versioning` variable i modulen som gjør versioning optional?
- Hint: Bruk `count` eller `for_each` basert på variabelen

```hcl
resource "aws_s3_bucket_versioning" "website" {
  count  = var.enable_versioning ? 1 : 0
  bucket = aws_s3_bucket.website.id
  # ...
}
```

---

## Del 3: CloudFront CDN

### Hvorfor CloudFront?

S3 website hosting har begrensninger:
- Ingen HTTPS-støtte
- Ikke globalt distribuert (treg for brukere langt fra bucket-regionen)
- Ingen custom domain uten ekstra oppsett

CloudFront løser disse problemene.

### Legg til CloudFront Distribution

**Utvid** `modules/s3-website/main.tf` med CloudFront:

```hcl
# ============================================
# CloudFront Distribution for Global CDN
# ============================================

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

### Oppdater Root Outputs

**I root `main.tf`**, legg til CloudFront output:

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

**Merk**: CloudFront deployment tar 5-15 minutter.

### Test CDN

```bash
terraform output cloudfront_url
```

Åpne URL-en i nettleseren. Legg merke til:
- HTTPS fungerer automatisk
- URL-en er global (CloudFront, ikke region-spesifikk)

Med ~40 linjer kode har du global CDN med HTTPS.

---

## Oppsummering

Du har nå lært:

- **Remote State Management**: State-deling i team og CI/CD
- **Terraform-moduler**: Gjenbrukbar infrastruktur-kode
- **CloudFront CDN**: Global distribusjon med HTTPS

**Neste steg**: Bonusoppgavene nedenfor dekker GitHub Actions CI/CD, custom domains, og flere Terraform-konsepter.

---

## Bonusoppgaver

### 1. Custom Domain med Data Sources

#### Hva er Data Sources?

Så langt har vi kun brukt `resource`-blokker, som oppretter nye ressurser i AWS. En **data source** leser informasjon om eksisterende ressurser uten å endre dem:

- `resource` oppretter (write)
- `data` henter (read-only)

#### Steg 1: Hent eksisterende Hosted Zone

Vi har en delt Route53 hosted zone for domenet `thecloudcollege.com` som du kan bruke. I stedet for å opprette en ny hosted zone, skal vi **hente** den eksisterende med en data source.

**Legg til øverst i `modules/s3-website/main.tf`**:

```hcl
# Data source - henter informasjon om eksisterende hosted zone
data "aws_route53_zone" "main" {
  zone_id = "Z09151061LZNRB9E4BYEL"  # thecloudcollege.com
}
```

#### Steg 2: Hent wildcard ACM-sertifikat fra us-east-1

Vi har et wildcard-sertifikat (`*.thecloudcollege.com`) i `us-east-1` som dekker alle subdomener.

**Legg til i `modules/s3-website/main.tf`**:

```hcl
# Data source - henter eksisterende wildcard ACM-sertifikat
# Bruker us-east-1 provider fordi CloudFront krever sertifikat i denne regionen
data "aws_acm_certificate" "wildcard" {
  provider = aws.us-east-1
  domain   = "*.thecloudcollege.com"
  statuses = ["ISSUED"]
}
```

#### Steg 3: Oppdater CloudFront til å bruke custom domain

Nå må CloudFront konfigureres til å akseptere requests fra ditt custom domain og bruke ACM-sertifikatet for HTTPS.

**Finn `aws_cloudfront_distribution` ressursen** i `modules/s3-website/main.tf` og gjør følgende endringer:

1. **Legg til `aliases` for custom domain** (rett under `enabled` og `default_root_object`):

```hcl
resource "aws_cloudfront_distribution" "website" {
  enabled             = true
  default_root_object = "index.html"
  aliases             = ["${var.subdomain}.thecloudcollege.com"]  # NYTT: Custom domain

  # ... resten av konfigurasjonen ...
}
```

2. **Erstatt `viewer_certificate` blokken** (den eksisterende bruker `cloudfront_default_certificate = true`):

```hcl
  viewer_certificate {
    acm_certificate_arn      = data.aws_acm_certificate.wildcard.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }
```

#### Steg 4: Opprett DNS-record

**Legg til Route53 record** i `modules/s3-website/main.tf`:

```hcl
# Resource - oppretter DNS-record som peker til CloudFront
resource "aws_route53_record" "website" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "${var.subdomain}.thecloudcollege.com"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.website.domain_name
    zone_id                = aws_cloudfront_distribution.website.hosted_zone_id
    evaluate_target_health = false
  }
}
```

#### Steg 5: Legg til subdomain-variabel

**Legg til i `modules/s3-website/variables.tf`**:

```hcl
variable "subdomain" {
  description = "Subdomain for the website (e.g., 'glenn' becomes glenn.thecloudcollege.com)"
  type        = string
}
```

**Oppdater modul-kallet i root `main.tf`**:

```hcl
module "s3_website" {
  source = "./modules/s3-website"

  bucket_name         = "ditt-bucket-navn"
  subdomain           = "ditt-navn"  # Endre til ditt navn

  tags = {
    Name        = "My Website"
    Environment = "Demo"
  }
}
```

#### Steg 6: Oppdater modul-kallet og outputs

**Oppdater modul-kallet i rot `main.tf`** for å inkludere subdomain og sende providers til modulen (se [Appendix A](#appendix-a-provider-configuration-i-moduler) for detaljer):

```hcl
module "s3_website" {
  source = "./modules/s3-website"

  # Send begge providers til modulen
  providers = {
    aws           = aws
    aws.us-east-1 = aws.us-east-1
  }

  bucket_name         = "ditt-bucket-navn"
  subdomain           = "ditt-navn"  # Endre til ditt unike navn (f.eks. "glenn")

  tags = {
    Name        = "My Website"
    Environment = "Demo"
  }
}
```

**Legg til output i rot `main.tf`**:

```hcl
output "custom_domain_url" {
  value       = module.s3_website.custom_domain_url
  description = "Custom domain URL with HTTPS"
}
```

**Legg til output i `modules/s3-website/outputs.tf`**:

```hcl
output "custom_domain_url" {
  description = "Your custom domain URL"
  value       = "https://${var.subdomain}.thecloudcollege.com"
}
```

**Opprett `modules/s3-website/versions.tf`** for å deklarere forventede providers:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
      configuration_aliases = [aws.us-east-1]
    }
  }
}
```

#### Steg 7: Deploy og test

1. **Re-initialiser Terraform** (nødvendig pga. ny provider):

```bash
terraform init
```

2. **Kjør plan** for å se hva som vil bli opprettet/endret:

```bash
terraform plan
```

Du skal se at CloudFront blir oppdatert og at en ny Route53 record blir opprettet.

3. **Apply endringene**:

```bash
terraform apply
```

**Merk**: CloudFront deployment tar 5-15 minutter når konfigurasjonen endres.

4. **Hent din custom domain URL**:

```bash
terraform output custom_domain_url
```

5. **Test din custom domain**:

Vent noen minutter på at CloudFront-distribusjonen er ferdig deployet, og åpne URL-en i nettleseren. Siden er nå tilgjengelig på `https://ditt-navn.thecloudcollege.com` med HTTPS.

**Nøkkelpunkter**:
- **Data sources** lar deg hente informasjon om eksisterende ressurser uten å endre dem
- **Provider alias** (`provider = aws.us-east-1`) lar deg bruke flere regioner i samme konfigurasjon
- CloudFront krever ACM-sertifikater i us-east-1 region
- Route53 `alias` records peker til AWS-ressurser (som CloudFront) uten å bruke IP-adresser
- Data sources refereres med `data.<type>.<name>`, f.eks. `data.aws_route53_zone.main.zone_id`

---

### 2. GitHub Actions CI/CD Pipeline

#### Mål

Automatiser Terraform deployment:
- **Pull Request**: Kjør `terraform plan` og vis endringer
- **Merge til main**: Kjør `terraform apply` automatisk

#### Steg 1: Opprett Workflow Fil

**Lag** `.github/workflows/terraform.yml`:

```yaml
name: Terraform Infrastructure

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  AWS_REGION: eu-west-1
  TF_VERSION: 1.6.0

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
        run: terraform plan -no-color
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
        run: terraform apply -auto-approve
```

#### Steg 2: Konfigurer GitHub Secrets

Du må gi GitHub Actions tilgang til AWS:

1. **Gå til ditt GitHub repository**
2. **Settings** → **Secrets and variables** → **Actions**
3. **Klikk "New repository secret"**
4. **Legg til to secrets**:
   - Name: `AWS_ACCESS_KEY_ID`, Value: `<din AWS access key>`
   - Name: `AWS_SECRET_ACCESS_KEY`, Value: `<din AWS secret key>`

**Sikkerhetstips**: Disse secrets bør være fra en dedicated IAM-bruker med minimal permissions (kun det Terraform trenger).

#### Steg 3: Test Pipeline

1. **Lag en ny branch**:

```bash
git checkout -b test-pipeline
```

2. **Gjør en liten endring** (f.eks. i README eller legg til en tag):

```hcl
# I main.tf
module "s3_website" {
  # ...
  tags = {
    # ...
    PipelineTest = "true"  # Ny tag
  }
}
```

3. **Commit og push**:

```bash
git add .
git commit -m "Test GitHub Actions pipeline"
git push origin test-pipeline
```

4. **Opprett Pull Request** på GitHub

5. **Observer**:
   - GitHub Actions kjører `terraform plan`
   - En kommentar vises på PR med plan output
   - Du kan se hva som vil endres før merge

6. **Merge PR** til main:
   - GitHub Actions kjører `terraform apply` automatisk
   - Infrastrukturen oppdateres uten manuell intervensjon

Du har nå full CI/CD for infrastrukturen din.

---

### 3. Validation Rules på variabler i modulen

Legg til validation i `modules/s3-website/variables.tf`:

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

---

## Ressurser

- [Terraform Modules Documentation](https://developer.hashicorp.com/terraform/language/modules)
- [AWS CloudFront Developer Guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/)
- [GitHub Actions Terraform Tutorial](https://developer.hashicorp.com/terraform/tutorials/automation/github-actions)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)

---

# Appendix

## Appendix A: Provider Configuration i Moduler

Denne seksjonen forklarer hvordan Terraform håndterer providers i moduler, spesielt når flere AWS-regioner er i bruk.

### Hvor skal providers konfigureres?

**Best practice**: Provider-konfigurasjon skal være i **hovedmodul**, ikke i undermoduler.

```hcl
# Hovedmodul (providers.tf) - RIKTIG
provider "aws" {
  region = "eu-west-1"
}

# Undermodul - IKKE konfigurer providers her
```

**Hvorfor?**
- Moduler skal være gjenbrukbare på tvers av ulike AWS-kontoer og regioner
- Hovedmodulen kontrollerer hvilke credentials og regioner som brukes
- Unngår konflikter når modulen brukes flere ganger

### Provider Inheritance

Som standard arver moduler automatisk provider-konfigurasjonen fra hovedmodulen:

```hcl
# Hovedmodul
provider "aws" {
  region = "eu-west-1"
}

module "s3_website" {
  source = "./modules/s3-website"
  # Provider arves automatisk - ingen ekstra konfigurasjon nødvendig
}
```

Dette fungerer for enkle tilfeller der du kun trenger én provider-konfigurasjon.

### Multi-Region Setup: Aliased Providers

I denne oppgaven trenger vi **to AWS providers** fordi:
- Hovedressurser (S3, CloudFront) skal være i `eu-west-1`
- ACM-sertifikater for CloudFront **må** være i `us-east-1` (AWS-krav)

#### Steg 1: Definer providers i hovedmodul

**Opprett eller oppdater `providers.tf` i rotmappen**:

```hcl
terraform {
  required_version = ">= 1.0"

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

# Alias provider for us-east-1
# Nødvendig fordi CloudFront krever ACM-sertifikater i us-east-1
provider "aws" {
  alias  = "us-east-1"
  region = "us-east-1"
}
```

**Forklaring**:
- Første `provider "aws"` uten alias er default provider
- Andre `provider "aws"` med `alias = "us-east-1"` er en navngitt provider
- Du kan ha flere aliased providers hvis du trenger flere regioner

#### Steg 2: Send providers til modulen

Når du bruker aliased providers, må du eksplisitt sende dem til modulen:

```hcl
# Hovedmodul (main.tf)
module "s3_website" {
  source = "./modules/s3-website"

  # Send begge providers til modulen
  providers = {
    aws           = aws           # Default provider
    aws.us-east-1 = aws.us-east-1 # Aliased provider
  }

  bucket_name = var.bucket_name
  subdomain   = var.subdomain
}
```

**Merk**:
- `aws = aws` sender default provider
- `aws.us-east-1 = aws.us-east-1` sender aliased provider
- Uten `providers`-blokken vil modulen kun få default provider

#### Steg 3: Deklarer forventede providers i modulen

Modulen må eksplisitt deklarere at den forventer en aliased provider:

```hcl
# modules/s3-website/versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
      configuration_aliases = [aws.us-east-1]
    }
  }
}
```

**`configuration_aliases`** forteller Terraform:
- Denne modulen forventer å motta en aliased provider kalt `aws.us-east-1`
- Hovedmodulen må sende denne provideren når modulen kalles

#### Steg 4: Bruk providers i ressurser

```hcl
# modules/s3-website/main.tf

# Bruker default provider (eu-west-1) - ingen provider-attributt nødvendig
resource "aws_s3_bucket" "website" {
  bucket = var.bucket_name
}

resource "aws_cloudfront_distribution" "website" {
  # Også default provider (eu-west-1)
  enabled = true
  # ...
}

# Bruker eksplisitt aliased provider (us-east-1)
data "aws_acm_certificate" "wildcard" {
  provider = aws.us-east-1  # Eksplisitt spesifisert
  domain   = "*.thecloudcollege.com"
  statuses = ["ISSUED"]
}
```

**Regel**:
- Ressurser **uten** `provider`-attributt bruker default provider
- Ressurser **med** `provider = aws.us-east-1` bruker aliased provider

### Vanlige Feil og Løsninger

#### Feil 1: "Provider configuration not present"

```
Error: Provider configuration not present
Module module.s3_website does not declare a provider named aws.us-east-1
```

**Løsning**: Legg til `configuration_aliases` i `modules/s3-website/versions.tf`.

#### Feil 2: "Module does not support aws.us-east-1"

```
Error: Module does not support aws.us-east-1 provider configuration
```

**Løsning**: Modulen må eksplisitt deklarere at den forventer aliased provider via `configuration_aliases`.

#### Feil 3: Ressurs bruker feil region

**Problem**: ACM-sertifikat blir opprettet i eu-west-1 i stedet for us-east-1.

**Løsning**: Legg til `provider = aws.us-east-1` på ressursen.

### Oppsummering

**For å bruke multi-region providers i moduler**:

1. **Hovedmodul**: Definer alle providers (default + aliased) i `providers.tf`
2. **Module call**: Send providers eksplisitt via `providers = { ... }`
3. **Module declaration**: Deklarer forventede providers med `configuration_aliases`
4. **Resources**: Bruk `provider = aws.alias` på ressurser som trenger aliased provider

Dette gir deg full kontroll over hvilke regioner som brukes for hvilke ressurser.
