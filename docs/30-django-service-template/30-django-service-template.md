---
title: "Django Service Template"
created: "2026-02-15"
updated: "2026-02-15"
version: 0.1.0
status: draft
type: reference
owner: "platform-team"
lifecycle: production
tags: [service, django, template, copier, allauth, rbac]
---

# Django Service Template — Copier-Generated Services

Updatable Copier template that generates production-ready Django services. Ships with API (DRF + HackSoft Styleguide 2+), authentication (django-allauth + Keycloak OIDC), authorization (RBAC with Keycloak as source of truth), and core (admin panel, healthchecks) apps. Optional public app with django-htmx frontend.

> 📍 **Type:** Service Reference
> 👤 **Owner:** Ktwenty Threel
> 📝 **Outcome:** Draft

---

## Placeholder

> [!warning] This document is a placeholder
> Content pending. Requires: service compose block, project directory structure, `settings.py`, `CustomSocialAccountAdapter`, RBAC sync implementation, and Copier template configuration.

### Planned Sections

- [[#Overview]] — What the template generates, how services register with infrastructure
- [[#Architecture]] — App structure (API, auth, authorization, core, public)
- [[#Design Decisions]] — HackSoft Styleguide, allauth over manual OIDC, RBAC sync strategy
- [[#Configuration]] — Compose block, environment variables, Django settings
- [[#Authentication Flow]] — allauth + Keycloak OIDC, backchannel token exchange, auto-linking
- [[#Authorization / RBAC]] — Group-to-role sync, `pre_social_login()`, permission model
- [[#Creating a New Service]] — Copier generation steps, Envoy route + Keycloak client registration
- [[#Dev ↔ Prod Differences]]
- [[#Runbooks]]
- [[#ADRs]]

---

## References

- [[infrastructure-engineering-hub|Infrastructure Engineering Hub]]
- [[service-keycloak|Keycloak Configuration]]
- [[service-envoy-gateway|Envoy Gateway Configuration]]
- [[service-postgres|PostgreSQL Configuration]]
- [[creating-new-service|Create a New Service]]
- [[template-customization|Template Customization]]
- [[app-api-hacksoft-integration|API App]]
- [[app-authentication-allauth-oidc|Authentication App]]
- [[app-authorisation-rbac|Authorization App]]
- [[app-core-admin-panel|Core App]]
- [[app-public-htmx-frontend|Public Frontend App]]