# Installation de l'Agent d'Outreach Instagram : Guide Technique

*[Deep Dive Tutorial - Daxios Field Notes]*

**Analyse technique :** Voici le guide d'installation complet de l'agent d'automatisation Instagram analysé dans notre précédent article. Configuration étape par étape avec tous les détails techniques.

---

## ⚠️ Cadre Légal et Technique

### Conformité RGPD
L'agent scrappe uniquement des **données publiques** d'Instagram :
- Pseudonymes utilisateurs (handles)
- Biographies publiques
- Nombre d'abonnés affiché publiquement
- Aucune donnée personnelle privée collectée

Ces informations étant publiquement accessibles, elles respectent le cadre RGPD pour le traitement de données.

### Violation des ToS Instagram
L'automatisation massive d'interactions viole explicitement les conditions d'utilisation d'Instagram. Risques techniques :
- Suspension temporaire ou permanente du compte
- Limitation des fonctionnalités (shadowban)
- Blocage IP en cas d'activité détectée comme non-humaine

---

## 🏗️ Architecture Technique

### Stack Technologique
- **N8N** : Orchestrateur de workflow pour la logique métier
- **Axiom.ai** : Engine d'automation browser
- **Parser.im** : API de scraping Instagram via proxies
- **Google Sheets** : Base de données CRM
- **Telegram Bot** : Interface de commande et notifications

### Flux de Données
```
Commande Telegram → N8N → Parser.im → Filtrage → IA → Google Sheets → Axiom → Instagram
```

---

## 📋 Prérequis Techniques

### Comptes et Services Requis
- N8N Cloud ou self-hosted
- Axiom.ai (plan Pro minimum)
- Parser.im avec API key
- Google Cloud Platform (pour Sheets API)
- Bot Telegram configuré

### APIs et Credentials
- OpenRouter API pour les modèles IA
- Google Sheets OAuth2 credentials
- Telegram Bot Token
- Parser.im API key (paiement crypto requis)

---

## 🔧 Configuration N8N

### 1. Import du Workflow

Créer un nouveau workflow et importer le JSON suivant (structure simplifiée) :

```json
{
  "name": "Cold Instagram Outreach Agent",
  "nodes": [
    {
      "type": "n8n-nodes-base.telegramTrigger",
      "name": "Telegram Trigger"
    },
    {
      "type": "@n8n/n8n-nodes-langchain.informationExtractor",
      "name": "EXTRACT VARIABLES FROM QUERY"
    },
    {
      "type": "n8n-nodes-base.set",
      "name": "SET VARIABLES"
    },
    {
      "type": "n8n-nodes-base.code",
      "name": "CREATE GET REQUEST URL"
    },
    {
      "type": "n8n-nodes-base.httpRequest",
      "name": "SEND SCRAPING REQUEST TO PARSER.IM"
    }
  ]
}
```

### 2. Configuration des Variables

Dans le node "SET VARIABLES" :
```javascript
{
  "IG": "{{ $json.output.IG }}",
  "API": "YOUR_PARSER_IM_API_KEY",
  "Limit": "{{ $json.output.Limit }}",
  "Keywords": "Coach, Mentor, Instructor, Expert",
  "Minimum subscribers": "5000",
  "Maximum Subscribers": "",
  "Exact match": false
}
```

### 3. Code de Génération d'URL Parser.im

```javascript
// Node: CREATE GET REQUEST URL
const ig = $input.first().json.IG;
const apiKey = $input.first().json.API;
const limit = $input.first().json.Limit;

const apiUrl = "https://parser.im/api.php";
const queryParams = {
  key: apiKey,
  mode: "create",
  type: "p1",
  act: "1",
  name: `${ig} ${limit}`,
  links: ig,
  spec: "2",
  limit: limit
};

const queryString = Object.entries(queryParams)
  .map(([key, value]) => `${encodeURIComponent(key)}=${encodeURIComponent(value)}`)
  .join("&");

return { url: `${apiUrl}?${queryString}` };
```

### 4. Logique de Filtrage

```javascript
// Node: CREATE FILTERING REQUEST URL
const apiKey = $('SET VARIABLES').first().json.API;
const bioKeyword = $('SET VARIABLES').first().json.Keywords || "";
const minSubscribers = $('SET VARIABLES').first().json['Minimum subscribers'];
const taskId = $('SEND SCRAPING REQUEST TO PARSER.IM').first().json.tid;

const formattedKeywords = bioKeyword.replace(/\s+/g, "");
const taskName = `Filter-${ig}-BIO ${formattedKeywords}-MinSubscribers ${minSubscribers || 0}`;

const queryParams = {
  key: apiKey,
  mode: "create",
  type: "f1",
  name: taskName,
  links: taskId,
  spec: "2",
  dop: "3,6,8",
  private: "1",
  white_bio: formattedKeywords,
  white_bio_match: "1"
};

if (minSubscribers !== undefined) {
  queryParams.followers1 = minSubscribers;
}

// Construction URL finale...
```

---

## 📊 Structure Google Sheets

### Feuille "CRM"
| Colonne A | Colonne B | Colonne C | Colonne D |
|-----------|-----------|-----------|-----------|
| Handle | URL | Message | Contacted |
| @username | https://instagram.com/@username | Message IA | NO |

### Feuille "DAILY OUTREACH"
Structure identique pour les prospects à contacter quotidiennement.

### Configuration Sheets API
```javascript
// Node: ADD PROSPECTS TO CRM
{
  "operation": "appendOrUpdate",
  "documentId": "YOUR_SHEET_ID",
  "sheetName": "CRM",
  "columns": {
    "Handle": "{{ $('ORGANISE DATA').item.json.handle }}",
    "URL": "=https://instagram.com/{{ $('ORGANISE DATA').item.json.handle }}",
    "Message": "{{ $json.output }}",
    "Contacted": "NO"
  }
}
```

---

## 🤖 Configuration Axiom.ai

### 1. Structure du Bot Browser

Import du template Axiom :
```json
{
  "name": "WORKING V3 DM IG NICK",
  "steps": [
    {
      "name": "Read data from Google Sheet",
      "type": "AxiomApiReadGoogleSheetWithRangeV3170"
    },
    {
      "name": "Loop through data",
      "type": "DriverNoUrl"
    },
    {
      "name": "Go to page",
      "url": "[google-sheet-data?*&1]"
    },
    {
      "name": "Click Message button",
      "selector": ".x1i10hfl.xjqpnuy.xa49m3k"
    },
    {
      "name": "Enter Text",
      "text": "[google-sheet-data?*&2]"
    }
  ]
}
```

### 2. Sélecteurs CSS Instagram

**Bouton Message :**
```css
.x1i10hfl.xjqpnuy.xa49m3k.xqeqjp1.x2hbi6w.x972fbf.xcfux6l
```

**Zone de texte DM :**
```css
textarea[aria-label*="Message"]
```

**Note :** Les sélecteurs CSS changent régulièrement. Surveillance requise.

### 3. Gestion des Erreurs

```javascript
// Vérification bouton Message disponible
if (document.querySelector('[aria-label*="Message"]')) {
  // Continuer l'automation
} else {
  // Passer au profil suivant
  return 'skip';
}
```

---

## 🔄 Workflow Complet

### 1. Commandes Telegram

**Génération de leads :**
```
Give me 5000 leads of competitor_handle
```

**Outreach quotidien :**
```
#daily
```

### 2. Séquence d'Exécution

1. **Phase Scraping** (1-3h selon limite)
   - Extraction des followers via Parser.im
   - Filtrage par critères (bio, abonnés)
   - Organisation des données

2. **Phase IA** (10-30 min)
   - Analyse de chaque profil
   - Génération message personnalisé
   - Stockage en CRM

3. **Phase Outreach** (selon rythme configuré)
   - Lecture Google Sheets par Axiom
   - Navigation automatique Instagram
   - Envoi messages + update statut

### 3. Monitoring et Logs

- Notifications Telegram à chaque étape
- Logs d'erreur détaillés
- Métriques de performance (taux succès/échec)

---

## ⚙️ Paramètres d'Optimisation

### Limites de Sécurité

**Comptes récents (< 6 mois) :**
- 20-30 messages/jour maximum
- Délai 2-5 minutes entre messages
- Pas d'activité weekend/soirée

**Comptes établis (> 1 an) :**
- 50-100 messages/jour maximum
- Délai 1-3 minutes entre messages
- Répartition sur créneaux naturels

### Code de Délais Aléatoires

```javascript
// Dans Axiom - Wait step
{
  "waitType": "random",
  "minDuration": "60000",
  "maxDuration": "180000"
}
```

---

## 📈 Métriques et Performance

### KPIs Techniques
- Taux de succès scraping : >95%
- Taux de livraison messages : 85-90%
- Temps moyen par message : 45-60 secondes
- Erreurs détectées : <5%

### Indicateurs Business
- Taux d'ouverture estimé : 70-80%
- Taux de réponse moyen : 2-5%
- Coût par lead qualifié : 0,50-1€
- ROI break-even : 1 client pour 2000 contacts

---

## 💰 Coûts Opérationnels

### Abonnements Mensuels
- **Parser.im :** 35$ (paiement crypto)
- **Axiom.ai Pro :** 99$
- **OpenRouter API :** 15-25$ selon usage
- **N8N Cloud :** 20$ (ou gratuit self-hosted)

**Total mensuel :** 170-180$

### Coûts de Setup Initial
- Configuration technique : 4-8h
- Tests et ajustements : 2-4h
- Formation utilisateur : 1-2h

---

## 🎯 Conclusion Technique

Ce système d'automation Instagram représente une prouesse technique intéressante combinant scraping, IA et browser automation. La complexité d'implémentation est réelle et nécessite des compétences techniques solides.

**Points forts techniques :**
- Architecture modulaire et scalable
- Gestion des erreurs et retry automatique
- Personnalisation IA des messages
- Monitoring temps réel

**Limitations opérationnelles :**
- Maintenance constante des sélecteurs CSS
- Dépendance à Parser.im (plateforme externe)
- Risque de détection par Instagram
- ROI difficile à prévoir selon le secteur
