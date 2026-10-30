# Plan: eerste workload, laag risico (2026-10-29)

Doel: de volledige SPIFFE-keten (registratie → node-attestatie → SVID-
uitgifte → bestand op schijf) end-to-end bewijzen, zonder een bestaande
productie-app aan te raken. Pas daarna, als los vervolg, een echte
workload (OpenProject-lab's DB-verbinding) omzetten.

Context: vandaag voor het eerst gedeployed op de echte ITlusions-cluster
(`itl`-context, OIDC-auth). Upstream's brede, automatische
`default`-registratie (98 entries voor bestaande productie-workloads) is
al uitgezet (zie `values.yaml` - `controllerManager.identities.
clusterSPIFFEIDs.default.enabled: false`) - vanaf nu krijgt alleen wat
expliciet in `registrationEntries` staat een identiteit.

## Stap 1: wegwerp-testpod, geen echte app

1. Een los, tijdelijk Deployment/Pod aanmaken in een eigen test-namespace
   (bijvoorbeeld `spire-poc`), met:
   - De `spiffe-helper`-sidecar (image `ghcr.io/spiffe/spiffe-helper`,
     zelfde image dat de oidc-discovery-provider al gebruikt).
   - De SPIFFE Workload API-socket gemount (via de `spiffe-csi-driver`,
     al aanwezig op elke node - zelfde mechanisme als `spire-agent` zelf
     gebruikt).
   - Een gedeeld volume (`emptyDir`) waar `spiffe-helper` het SVID
     (`cert.pem`, `key.pem`, `bundle.pem`) naartoe schrijft.
2. Eén `registrationEntries`-item toevoegen aan `values.yaml`, specifiek
   voor deze testpod's namespace/serviceaccount - niets breder.
3. `helm upgrade` draaien zodat de nieuwe entry actief wordt.

## Stap 2: verifiëren

1. `kubectl exec` in de testpod, bevestigen dat `cert.pem`/`key.pem`
   daadwerkelijk bestaan en een geldig X.509-certificaat met een
   `spiffe://itlusions.local/...`-URI als SAN bevatten
   (`openssl x509 -in cert.pem -noout -text | grep -A1 "Subject Alternative Name"`).
2. Rotatie bevestigen: het certificaat na verloop van tijd (standaard
   binnen een uur) laten vernieuwen, en controleren dat het bestand
   daadwerkelijk verandert (nieuwe `NotBefore`-timestamp).
3. Opruimen: de testpod en de bijbehorende `registrationEntries`-regel
   verwijderen zodra dit bevestigd is - dit is een PoC, geen permanente
   resource.

## Stap 3, apart vervolg (niet in deze sessie): OpenProject-lab

Pas na een succesvolle stap 1/2:

1. Een `registrationEntries`-item voor `openproject-lab`'s
   serviceaccount(s), gericht op de echte workload (niet breed via een
   namespace-wildcard).
2. `spiffe-helper`-sidecar aan de OpenProject-pod zelf toevoegen (of aan
   de CNPG-cluster-pod, afhankelijk van welke kant het certificaat moet
   presenteren).
3. CloudNativePG's clientauthenticatie omzetten van wachtwoord naar
   certificaat-gebaseerd, voor deze ene verbinding - niet clusterbreed.
4. Pas als dit stabiel draait: overwegen of andere workloads (Matrix,
   Nextcloud, Forgejo) dezelfde aanpak krijgen.

## Risico's en afbakening

- Stap 1/2 raakt geen bestaande app, geen bestaande data - volledig
  reversibel door de testpod + registration entry te verwijderen.
- Stap 3 raakt een lab-omgeving, niet productie - nog steeds relatief
  laag risico, maar wel een echte, bestaande workload.
- Niet in scope van dit plan: Matrix, Nextcloud, Forgejo, of enige
  productie-kritieke workload (Longhorn, MetalLB, CNPG-operator zelf) -
  die komen pas aan de orde als stap 3 bewezen stabiel is.
