# 🚀 HOTEL AGENT - META ADS AUTOMATION MASTER PROMPT

## PROJE KİMLİĞİ VE BAĞLAM

**Şirket:** EtsTur Turizm A.Ş. - Hotel Agent Markası  
**Ekip:** 5 dijital pazarlama uzmanı (batuhan.celik@etstur.com, mert.pektas@etstur.com, ece.sarac@etstur.com)  
**Teknoloji Stack:** n8n (Self Hosted v2.0.3) + Google Sheets + Meta Marketing API v23.0  
**Workflow Adı:** Hotel Agent - Automation v20 (ULTIMATE FIX)  
**Workflow ID:** JM5MUpPGVYxNBdph

---

## 🎯 PROJENİN ANA HEDEFİ

Dijital pazarlama ekibinin Meta Ads Manager'a **hiç girmeden** tüm Facebook & Instagram reklam operasyonlarını **sadece Google Sheets üzerinden** yönetmesini sağlayan tam otomatik sistem.

### İş Problemi
Ekip üyeleri günlerinin büyük bölümünü Meta Ads Manager'da manuel kampanya oluşturma/düzenleme ile geçiriyor. Manuel değişiklikler Google Sheets ile senkronize olmadığında veri tutarsızlıkları oluşuyor.

### Çözüm Mimarisi
```
Google Sheets (Veri Girişi) 
    ↓ [30 dk polling]
n8n Workflow (İşleme)
    ↓ [API Calls]
Meta Marketing API v23.0 (Kampanya Yönetimi)
    ↓ [ID Geri Yazma]
Google Sheets (Sonuç Güncelleme)
```

---

## 📊 MEVCUT WORKFLOW YAPISI (66 NODE)

### Tetikleyiciler (Triggers)
1. **Meta Ads Automation** - Google Sheets Trigger (30 dk polling, A2:AZ20000 range)
2. **Schedule Trigger** - Sync modülü için dakikalık tetikleyici

### Ana Modüller

#### 1️⃣ CREATE MODULE (Kampanya/AdSet/Ad Oluşturma)
```
Action = "create_campaign" tetikler

Akış (CBO - Campaign Budget Optimization):
Meta Ads Automation → Input Validation → Mapping → Date & Time → Action Router
    → If Budget Level (Camp Budget) [TRUE]
    → Build Campaign Body (CBO) → Camp Budget - Camp. Create
    → Camp. ID → Build Final Ad Set Body → Campaign Budget - Ad Set Create
    → Write Ad Set ID → Build Final Ad Body (CBO) → Campaign Budget - Ad Create
    → Mark Action NONE (CBO)

Akış (ABO - Ad Set Budget Optimization):
    → If Budget Level (Camp Budget) [FALSE]
    → Build Campaign Body (ABO) → Ad Set Budget - Camp. Create
    → Camp. ID1 → Build Final Ad Set Body1 → AD Set Budget - Ad Set Create
    → Write Ad Set ID1 → Build Final Ad Body (ABO) → Ad Set Budget - Ad Create
    → Mark Action NONE (ABO)
```

#### 2️⃣ UPDATE MODULE (Kampanya/AdSet/Ad Güncelleme)
```
Action = "update" tetikler

Akış (CBO):
Action Router [update] → If Budget Level (Camp Budget)1 [TRUE]
    → Build CBO Campaign Update Body → Camp Budget - Camp. update
    → Build Final Ad Set UPDATE Body (CBO) → Campaign Budget - Ad Set Update
    → Build CBO Ads Update → Campaign Budget - Ad Update (Ad Creatives)
    → Campaign Budget - Ad Update (New Ad Body)
    → If CBO (Ad Name - Status) → Campaign Budget - Ad Update (Ad ID)
    → Mark Action NONE (CBO)-Name/Status / Mark Action NONE (CBO) New Ad Body

Akış (ABO):
    → If Budget Level (Camp Budget)1 [FALSE]
    → Build ABO Campaign Update Body → Ad Set Budget - Camp. update
    → Build Final Ad Set UPDATE Body (ABO) → Ad Set Budget - Ad Set Update
    → Build ABO Ads Update → Ad Set Budget - Ad Update (Ad Creatives)
    → Ad Set Budget - Ad Update (New Ad Body)
    → If ABO (Ad Name - Status) → Ad Set Budget - Ad Update (Ad ID)
    → Mark Action NONE (ABO)-Name/Status / Mark Action NONE (ABO) New Ad Body
```

#### 3️⃣ SYNC MODULE (Meta → Google Sheets Senkronizasyon)
```
Sync = "sync" tetikler

Akış:
Action Router [sync] → Sync to Google Sheets → Code in JavaScript - Sync
    → Fetch from Meta → Code in JavaScript - Fetch from Meta
    → Code in JavaScript3 → Sync to Google Sheets - Fetch from Meta
    
Veya Schedule Trigger ile:
Schedule Trigger → Sync to Google Sheets → ...
```

---

## 📋 GOOGLE SHEETS YAPISI (43 SÜTUN)

### Sheet Bilgileri
- **Document ID:** 1MQlMgOSWr2eE8lWBUFRkPIwF8ilbiPACxzxkrYAK0ek
- **Sheet Name:** Son (GID: 222526350)
- **Range:** A2:AZ20000
- **Header Row:** 2, First Data Row: 3

### Kritik Sütunlar ve Mapping

```
KAMPANYA SEVİYESİ:
├── Account ID          → account_id
├── Campaign ID         → campaign_id (API'den geri yazılır)
├── Campaign Name       → campaign_name
├── Objective           → objective (OUTCOME_TRAFFIC, OUTCOME_LEADS, etc.)
├── Buying Type         → buying_type (AUCTION)
├── Special Ad Categories → special_ad_categories
├── Budget              → budget_type ("Campaign Budget" / "Ad Set Budget")
├── Lifetime Budget     → lifetime_budget
└── Status (Campaign)   → status (ACTIVE/PAUSED/DELETED)

AD SET SEVİYESİ:
├── Ad Set ID           → adset_id (API'den geri yazılır)
├── Ad Set Name         → adset_name
├── Ad Set Budget       → daily_budget (TRY, API'ye kuruş olarak gönderilir)
├── Start Date          → start_time (Excel serial → ISO 8601)
├── End Date            → end_time
├── Age_Min             → targeting.age_min (min: 13)
├── Age_Max             → targeting.age_max (max: 65)
├── Bid Strategy        → bid_strategy
├── Conversion Location → destination_type mapping
├── Optimization Goal   → optimization_goal
├── Platform            → publisher_platforms
├── Placements          → facebook_positions / instagram_positions
├── Billing Event       → billing_event
├── Status (Ad Set)     → status
└── Pixel_ID            → promoted_object.pixel_id

AD SEVİYESİ:
├── AD ID               → ad_id (API'den geri yazılır)
├── Ad Name             → name
├── Primary Text        → object_story_spec.link_data.message
├── Headline            → object_story_spec.link_data.name
├── Description         → object_story_spec.link_data.description
├── URL                 → object_story_spec.link_data.link
├── Call To Action      → call_to_action.type
├── Call Number         → call_to_action.value.link (tel:+90...)
├── Status (Ad)         → status
├── FB Page_ID          → object_story_spec.page_id
└── IG User_ID          → instagram_actor_id

KONTROL ALANLARI:
├── Sync                → "sync" olduğunda Meta'dan veri çeker
├── Action              → "create_campaign" / "update" / "none"
├── Processed           → İşlem durumu
├── Last Error          → Son hata mesajı
├── Updated At          → Son güncelleme tarihi
├── Warnings            → Uyarılar
└── Row No              → Satır numarası (matching için)
```

### Sheet'te Kullanılan Değerler

```
Action:
  - create_campaign  → Yeni kampanya/adset/ad oluşturur
  - update           → Mevcut kampanya/adset/ad günceller
  - none             → İşlem yapmaz (varsayılan)

Budget:
  - Campaign Budget  → CBO (Campaign Budget Optimization)
  - Ad Set Budget    → ABO (Ad Set Budget Optimization)

Conversion Location:
  - WEBSITE          → Website trafiği
  - CALLS            → Telefon aramaları
  - MESSAGING        → Mesajlaşma uygulamaları
  - ON_AD            → Reklam üzerinde (Lead Form)
  - NONE             → Tanımsız (HATA!)

Platform:
  - Facebook         → Sadece Facebook
  - Instagram        → Sadece Instagram
  - Both             → Her iki platform

Placements:
  - Facebook Feed, Facebook Stories, Facebook Reels
  - Instagram Feed, Instagram Stories, Instagram Reels
  - AUTO             → Otomatik yerleşim
```

---

## 🔴 TESPİT EDİLEN SORUNLAR VE ÇÖZÜMLER

### SORUN 1: billing_event / optimization_goal / destination_type Uyumsuzluğu
**Hata Kodu:** 1815117  
**Hata Mesajı:** "Billing event invalid for optimisation goal"

#### Meta API Uyumluluk Kuralları:

| destination_type | Geçerli optimization_goal | Geçerli billing_event | Platform Kısıtı |
|-----------------|---------------------------|----------------------|-----------------|
| WEBSITE | LINK_CLICKS | LINK_CLICKS, IMPRESSIONS | Facebook, Instagram |
| WEBSITE | CONVERSIONS | IMPRESSIONS | Facebook, Instagram |
| PHONE_CALL | QUALITY_CALL | IMPRESSIONS | **Sadece Facebook** |
| MESSAGING_APPS | CONVERSATIONS | IMPRESSIONS | Facebook, Instagram |
| ON_AD | LEAD_GENERATION | IMPRESSIONS | Facebook, Instagram |
| ON_AD | REACH | IMPRESSIONS | Facebook, Instagram |

#### Çözüm Kodu (Build Final Ad Set Body içine eklenmeli):
```javascript
function validateAndFixApiParams(conversionLocation, billingEvent, optimizationGoal, platform) {
  const destType = mapConversionToDestination(conversionLocation);
  
  // PHONE_CALL kuralları
  if (destType === 'PHONE_CALL') {
    return {
      destination_type: 'PHONE_CALL',
      billing_event: 'IMPRESSIONS',  // Zorunlu
      optimization_goal: 'QUALITY_CALL',  // Zorunlu
      platform: 'facebook',  // Sadece Facebook
      facebook_positions: ['feed'],  // Instagram YOK
      instagram_positions: []
    };
  }
  
  // MESSAGING kuralları
  if (destType === 'MESSAGING_APPS') {
    return {
      destination_type: 'MESSAGING_APPS',
      billing_event: 'IMPRESSIONS',
      optimization_goal: 'CONVERSATIONS',
      // Instagram Story/Feed kullanılamaz
    };
  }
  
  // WEBSITE için LINK_CLICKS kontrolü
  if (destType === 'WEBSITE' && optimizationGoal === 'LINK_CLICKS') {
    return {
      destination_type: 'WEBSITE',
      billing_event: billingEvent === 'LINK_CLICKS' ? 'LINK_CLICKS' : 'IMPRESSIONS',
      optimization_goal: 'LINK_CLICKS'
    };
  }
  
  return { destination_type: destType, billing_event, optimization_goal };
}

function mapConversionToDestination(convLoc) {
  const mapping = {
    'WEBSITE': 'WEBSITE',
    'CALLS': 'PHONE_CALL',
    'MESSAGING': 'MESSAGING_APPS',
    'ON_AD': 'ON_AD',
    'NONE': 'WEBSITE'  // Varsayılan
  };
  return mapping[convLoc.toUpperCase()] || 'WEBSITE';
}
```

---

### SORUN 2: Instagram Positions - "stream" vs "feed" Hatası
**Hata:** "Invalid feed value for the instagram_positions placement field"

#### Neden:
Meta API v23.0'da Instagram positions için `stream` değeri `feed` olarak değişti.

#### Çözüm:
```javascript
// Eski (YANLIŞ)
instagram_positions: ['stream', 'story', 'reels']

// Yeni (DOĞRU)
instagram_positions: ['feed', 'story', 'reels']
```

---

### SORUN 3: TRY → Kuruş Dönüşümü
**Meta API:** Bütçe kuruş (cent) cinsinden beklenir

#### Çözüm:
```javascript
function convertToKurus(tryAmount) {
  if (tryAmount === '' || tryAmount == null) return null;
  const num = parseFloat(tryAmount);
  if (isNaN(num) || num < 1) return null;
  const kurus = Math.round(num * 100);
  // Minimum 100 TRY = 10000 kuruş
  return kurus >= 10000 ? String(kurus) : '10000';
}
```

---

### SORUN 4: Excel Tarih → ISO 8601 Dönüşümü
**Sheet'te:** Excel serial number (örn: 45678)  
**API'de:** ISO 8601 format (örn: 2025-01-05T00:00:00Z)

#### Çözüm:
```javascript
function convertExcelDateToISO(excelDate) {
  if (!excelDate || excelDate === '') return '';
  if (typeof excelDate === 'string' && excelDate.includes('T')) return excelDate;
  const num = Number(excelDate);
  if (isNaN(num)) return '';
  const epochStart = Date.UTC(1899, 11, 30);
  const millisPerDay = 86400000;
  return new Date(epochStart + num * millisPerDay).toISOString();
}
```

---

### SORUN 5: Data Flow Kaybı ($itemIndex Matching)
**Problem:** HTTP Request node'ları sonrasında orijinal row bilgisi kayboluyor

#### Çözüm:
```javascript
// Her node'da $input.itemIndex kullanarak orijinal data'yı koru
const currentIndex = $input.itemIndex;
const originalData = $('Mapping').all()[currentIndex]?.json || {};
```

---

### SORUN 6: promoted_object Eksikliği
**Bazı destination_type'lar için zorunlu**

#### Çözüm:
```javascript
function buildPromotedObject(destType, pageId, pixelId) {
  switch (destType) {
    case 'PHONE_CALL':
      return { page_id: pageId };
    case 'WEBSITE':
      return pixelId ? { pixel_id: pixelId, custom_event_type: 'OTHER' } : null;
    case 'ON_AD':
      return { page_id: pageId };
    default:
      return null;
  }
}
```

---

### SORUN 7: Code Node'larda Eksik Validasyonlar

#### "Build Final Ad Body (CBO)" ve "Build Final Ad Body (ABO)" node'larında eksik kontroller:
- ❌ billing_event kontrolü YOK
- ❌ optimization_goal kontrolü YOK
- ❌ destination_type mapping YOK
- ❌ Phone call destination kontrolü YOK
- ❌ promoted_object oluşturma YOK
- ❌ Instagram positions düzeltmesi YOK
- ❌ TRY to kuruş dönüşümü YOK

---

## 🛠️ TAM DÜZELTME GEREKTİREN NODE'LAR

### 1. Build Final Ad Set Body (CBO için)
```javascript
/**
 * Build Final Ad Set Body - COMPLETE FIX
 * CBO Mode: Budget at campaign level, no daily_budget here
 */

function clean(v) {
  if (typeof v !== 'string') return v == null ? '' : String(v);
  return v.replace(/\u00A0/g, ' ').replace(/\r?\n/g, ' ').trim()
    .replace(/^="\s*|\s*"$/g, '');
}

function convertToKurus(tryAmount) {
  if (tryAmount === '' || tryAmount == null) return null;
  const num = parseFloat(tryAmount);
  if (isNaN(num) || num < 1) return null;
  return Math.max(Math.round(num * 100), 10000);
}

function convertExcelDateToISO(excelDate) {
  if (!excelDate || excelDate === '') return '';
  if (typeof excelDate === 'string' && excelDate.includes('T')) return excelDate;
  const num = Number(excelDate);
  if (isNaN(num)) return '';
  return new Date(Date.UTC(1899, 11, 30) + num * 86400000).toISOString();
}

function mapConversionToDestination(convLoc) {
  const mapping = {
    'WEBSITE': 'WEBSITE',
    'CALLS': 'PHONE_CALL',
    'MESSAGING': 'MESSAGING_APPS',
    'ON_AD': 'ON_AD'
  };
  return mapping[clean(convLoc).toUpperCase()] || 'WEBSITE';
}

function validateApiParams(destType, billingEvent, optGoal) {
  // PHONE_CALL zorunlu düzeltmeler
  if (destType === 'PHONE_CALL') {
    return {
      billing_event: 'IMPRESSIONS',
      optimization_goal: 'QUALITY_CALL',
      forceFacebookOnly: true
    };
  }
  
  // MESSAGING zorunlu düzeltmeler
  if (destType === 'MESSAGING_APPS') {
    return {
      billing_event: 'IMPRESSIONS',
      optimization_goal: 'CONVERSATIONS',
      forceFacebookOnly: false
    };
  }
  
  // Geçersiz billing_event düzeltmeleri
  const validBillingEvents = ['IMPRESSIONS', 'LINK_CLICKS', 'APP_INSTALLS', 'THRUPLAY'];
  if (!validBillingEvents.includes(billingEvent)) {
    return {
      billing_event: 'IMPRESSIONS',
      optimization_goal: optGoal,
      forceFacebookOnly: false
    };
  }
  
  return { billing_event: billingEvent, optimization_goal: optGoal, forceFacebookOnly: false };
}

function buildTargeting(row) {
  const ageMin = Math.max(13, Math.min(65, parseInt(clean(row.age_min || row['Age_Min'])) || 18));
  const ageMax = Math.max(ageMin, Math.min(65, parseInt(clean(row.age_max || row['Age_Max'])) || 65));
  
  return {
    geo_locations: { countries: ['TR'] },
    age_min: ageMin,
    age_max: ageMax
  };
}

function buildPublisherPlatforms(platform, forceFacebookOnly) {
  if (forceFacebookOnly) return ['facebook'];
  
  const p = clean(platform).toLowerCase();
  if (p.includes('both') || (p.includes('facebook') && p.includes('instagram'))) {
    return ['facebook', 'instagram'];
  }
  if (p.includes('instagram')) return ['instagram'];
  return ['facebook'];
}

function buildPositions(placements, publishers, forceFacebookOnly) {
  const fb = [];
  const ig = [];
  const placementStr = clean(placements).toLowerCase();
  
  if (placementStr === 'auto' || !placementStr) {
    return { facebook_positions: null, instagram_positions: null }; // Auto placement
  }
  
  // Facebook positions
  if (publishers.includes('facebook')) {
    if (placementStr.includes('facebook feed')) fb.push('feed');
    if (placementStr.includes('facebook stories')) fb.push('story');
    if (placementStr.includes('facebook reels')) fb.push('reels');
    if (fb.length === 0) fb.push('feed');
  }
  
  // Instagram positions (skip if PHONE_CALL)
  if (publishers.includes('instagram') && !forceFacebookOnly) {
    if (placementStr.includes('instagram feed')) ig.push('feed');  // NOT 'stream'!
    if (placementStr.includes('instagram stories')) ig.push('story');
    if (placementStr.includes('instagram reels')) ig.push('reels');
    if (ig.length === 0) ig.push('feed');
  }
  
  return {
    facebook_positions: fb.length > 0 ? fb : null,
    instagram_positions: ig.length > 0 ? ig : null
  };
}

function buildPromotedObject(destType, pageId, pixelId) {
  if (destType === 'PHONE_CALL' || destType === 'ON_AD') {
    return pageId ? { page_id: pageId } : null;
  }
  if (destType === 'WEBSITE' && pixelId) {
    return { pixel_id: pixelId, custom_event_type: 'OTHER' };
  }
  return null;
}

// MAIN EXECUTION
const row = item.json || {};
const prevNode = $('Camp. ID').all()[$input.itemIndex]?.json || {};
const campaignId = clean(prevNode.campaign_id || row.campaign_id || '');
const accountId = clean(row.account_id || row['Account ID'] || '');

// Get raw values
const adsetName = clean(row['Ad Set Name'] || row.adset_name || `AdSet-${Date.now()}`);
const status = clean(row['Status (Ad Set)'] || row.status_adset || 'PAUSED').toUpperCase();
const convLoc = clean(row['Conversion Location'] || row.conversion_location || 'WEBSITE');
const billingEvent = clean(row['Billing Event'] || row.billing_event || 'IMPRESSIONS').toUpperCase();
const optGoal = clean(row['Optimization Goal'] || row.optimization_goal || 'LINK_CLICKS').toUpperCase();
const platform = clean(row['Platform'] || row.platform || 'Facebook');
const placements = clean(row['Placements'] || row.placement || 'AUTO');
const bidStrategy = clean(row['Bid Strategy'] || row.bid_strategy || 'LOWEST_COST_WITHOUT_CAP').toUpperCase();
const pageId = String(clean(row['FB Page_ID'] || row.fb_page_id || '')).replace(/\D/g, '');
const pixelId = String(clean(row['Pixel_ID'] || row.pixel_id || '')).replace(/\D/g, '');

// Map and validate
const destType = mapConversionToDestination(convLoc);
const validated = validateApiParams(destType, billingEvent, optGoal);

// Build components
const publishers = buildPublisherPlatforms(platform, validated.forceFacebookOnly);
const positions = buildPositions(placements, publishers, validated.forceFacebookOnly);
const targeting = buildTargeting(row);
targeting.publisher_platforms = publishers;
if (positions.facebook_positions) targeting.facebook_positions = positions.facebook_positions;
if (positions.instagram_positions) targeting.instagram_positions = positions.instagram_positions;

const promotedObject = buildPromotedObject(destType, pageId, pixelId);

// Build final body
const finalBody = {
  name: adsetName,
  campaign_id: campaignId,
  status: ['ACTIVE', 'PAUSED', 'DELETED'].includes(status) ? status : 'PAUSED',
  billing_event: validated.billing_event,
  optimization_goal: validated.optimization_goal,
  bid_strategy: bidStrategy,
  destination_type: destType,
  targeting: targeting,
  start_time: convertExcelDateToISO(row['Start Date'] || row.start_time),
  end_time: convertExcelDateToISO(row['End Date'] || row.end_time)
};

// CBO: NO daily_budget at adset level
// Add promoted_object if exists
if (promotedObject) finalBody.promoted_object = promotedObject;

// Clean undefined/null values
Object.keys(finalBody).forEach(k => {
  if (finalBody[k] === '' || finalBody[k] == null) delete finalBody[k];
});

item.json._finalBody = finalBody;
item.json.account_id = accountId;
item.json.campaign_id = campaignId;
item.json._debug = {
  destType,
  validated,
  publishers,
  positions
};

return item;
```

### 2. Build Final Ad Set Body1 (ABO için)
ABO'da tek fark: `daily_budget` adset seviyesinde eklenir:
```javascript
// CBO kodunun aynısı + şu ekleme:
const dailyBudget = convertToKurus(row['Ad Set Budget'] || row.daily_budget);
if (dailyBudget) finalBody.daily_budget = dailyBudget;
```

---

## 📌 META API UPDATE EDİLEBİLİR ALANLAR

### Campaign UPDATE (Sadece bunlar değiştirilebilir):
- name
- status
- daily_budget (CBO)
- lifetime_budget

### Ad Set UPDATE (Sadece bunlar değiştirilebilir):
- name
- status
- daily_budget (ABO)
- start_time
- end_time

**⚠️ DEĞİŞTİRİLEMEZ:** billing_event, optimization_goal, destination_type, targeting, bid_strategy

### Ad UPDATE:
- name
- status
- creative (yeni ad creative oluşturulmalı)

---

## 🔗 API ENDPOINT'LERI (v23.0)

```
CREATE:
POST https://graph.facebook.com/v23.0/act_{account_id}/campaigns
POST https://graph.facebook.com/v23.0/act_{account_id}/adsets
POST https://graph.facebook.com/v23.0/act_{account_id}/adcreatives
POST https://graph.facebook.com/v23.0/act_{account_id}/ads

UPDATE:
POST https://graph.facebook.com/v23.0/{campaign_id}
POST https://graph.facebook.com/v23.0/{adset_id}
POST https://graph.facebook.com/v23.0/{ad_id}

READ (Sync için):
GET https://graph.facebook.com/v23.0/{ad_id}?fields=name,status,creative{object_story_spec,...}
```

---

## ✅ ÇÖZÜM UYGULAMA ADIMLARI

### Adım 1: Environment Variable
```
META_ACCESS_TOKEN = [your_token_here]
```

### Adım 2: Düzeltilmesi Gereken Code Node'lar
1. **Build Final Ad Set Body** - Tam yeniden yazılacak (yukarıdaki kod)
2. **Build Final Ad Set Body1** - Tam yeniden yazılacak (ABO versiyonu)
3. **Build Final Ad Body (CBO)** - destination_type kontrolü eklenecek
4. **Build Final Ad Body (ABO)** - destination_type kontrolü eklenecek
5. **Input Validation** - billing_event/optimization_goal çift kontrolü eklenecek

### Adım 3: Google Sheets Veri Düzeltmeleri
Geçersiz kombinasyonları düzeltin:
- CALLS + LINK_CLICKS → CALLS + IMPRESSIONS + QUALITY_CALL
- MESSAGING + LINK_CLICKS → MESSAGING + IMPRESSIONS + CONVERSATIONS
- REACH (Billing Event olarak) → IMPRESSIONS
- CLICKS → LINK_CLICKS veya IMPRESSIONS
- NONE (Conversion Location) → WEBSITE

### Adım 4: Test Senaryoları
1. CBO + WEBSITE + LINK_CLICKS → ✅ Çalışmalı
2. ABO + WEBSITE + LINK_CLICKS → ✅ Çalışmalı
3. CBO + CALLS + QUALITY_CALL → ✅ Sadece Facebook'ta çalışmalı
4. ABO + MESSAGING + CONVERSATIONS → ✅ Çalışmalı
5. CBO + ON_AD + LEAD_GENERATION → ✅ Çalışmalı

---

## 📚 REFERANSLAR

- Meta Marketing API: https://developers.facebook.com/docs/marketing-api
- Ad Set Create: https://developers.facebook.com/docs/marketing-api/get-started/basic-ad-creation/create-an-ad-set
- Optimization Goals: https://developers.facebook.com/docs/marketing-api/bidding/overview
- Billing Events: https://developers.facebook.com/docs/marketing-api/reference/ad-set/

---

## 🎯 BAŞARI KRİTERLERİ

1. ✅ Hiçbir API hatası olmadan kampanya oluşturma
2. ✅ CBO ve ABO modları doğru çalışıyor
3. ✅ Update modülü sadece değiştirilebilir alanları güncelliyor
4. ✅ Sync modülü Meta'dan Sheet'e doğru veri çekiyor
5. ✅ Tüm satırlar için Row No matching çalışıyor
6. ✅ Error handling ve Google Sheets comment bildirimleri çalışıyor

---
V21 json içerisindeki her node’u, google sheets (meta ads automation - sayfa:son) ’( girdilerini , Facebook developer sayfasını ve n8n mimarisini çok iyi öğrenmen lazım.  Yazdıklarını ve otomasyonla ilgili detayları cevaplayayım bir sonraki cevaplarında bu bilgileri öğrenmiş bir şekilde cevapla. Ayrıca bana hiçbir zaman kısa kodlar verip bunu x-x yerine ekle deme. Direkt hangi node içerisine ekleyeceğim tüm kodu vermen lazım. Bunun için ilk dosyadaki json dosyasından beslenebilirsin veya adım adım gideceğimiz için ben sana mevcut kodları veriririm onları düzelterek gideriz. Aşama aşama gitmek için sorunların yanına v1-v2 vs ekledim. eğer verdiğin cevaplar çözüm olmazsa onu çözene kadar bir sonraki soruna(V) ye geçmeyiz, orayı kusursuz hale getirmeliyiz!

SS1- Mevcut Campaign Budget - Ad Set Create hatası
SS2 - Camp/ Ad Set / Ads Create  (CBO/ABO) create ve update akışlarında bu yapıyı dikkate almalıyız.
SS3- Camp/ Ad Set / Ads Update (CBO/ABO) create ve update akışlarında bu yapıyı dikkate almalıyız.
SS4 - Sync
SS5- Sync - Google sheets yapısı

Camp/ Ad Set / Ads Create ( CBO = Campaign Budget (burada sadece kampanya düzeyinde bütçe veriyoruz) / ABO= Ad Set Budget/ (burada sadece ad set düzeyinde bütçe veriyoruz.

Camp/ Ad Set / Ads Update ( CBO = Campaign Budget (burada sadece kampanya düzeyinde update yapıyoruz) / ABO= Ad Set Budget/ (burada sadece ad set düzeyinde update yapıyoruz)

Sync = Burada yapı şöyle işliyor. Ekip dosya içerisinde create veya update gibi bir aksiyon almadan önce ss5 de bulunan google sheets “A” sütununda bulunan “Sync” başlığı altında update edeceği reklamı(satırı) belirliyor ve A sütunundaki sync butonunu işaretliyor. N8N workflowu her iki dakikada bir execute olduğundan dolayı dosyada butonu işaretli bir satır varsa sistem ilk önce metaya request atarak kampanyanın, ad set ve ads’in güncel bilgilerini çekiyor ve google sheets içerisinde ilgili hücreleri-girdileri güncelliyor.  Bu aksiyonla kampanyanın, ad set ve ads’in üzerine yazma sorunu çözmeyi hedefliyoruz.

	
