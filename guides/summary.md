# Alex - Agentic Learning Equities eXplainer (Yhteenveto)

Tämä dokumentti on korkean tason yleiskatsaus ja "arkkitehtuurinen muistilappu" Alex-projektille. Alex on yritystason (enterprise-grade) SaaS-sovellus talouden suunnitteluun, ja se rakennettiin Ed Donnerin *AI in Production* -kurssin viikoilla 3 ja 4.

Tämän dokumentin tarkoituksena on toimia referenssinä, jonka pohjalta voit myöhemmin rakentaa vastaavanlaisia hajautettuja tekoälyagenttijärjestelmiä AWS-pilveen. 

---

## 🏗 Projektin Arkkitehtuuri ja Teknologiat

Alex on moniagenttijärjestelmä (Multi-Agent System), joka hyödyntää täysin **Serverless** -arkkitehtuuria AWS-ympäristössä. Se kytkeytyy nykyaikaiseen React-käyttöliittymään.

### 1. Käyttöliittymä (Frontend)
- **Framework:** Next.js (React)
- **Autentikointi:** Clerk (JWT-validaatio taustajärjestelmässä)
- **Hosting:** Staattiset sivut Amazon S3 -ämpärissä ja CloudFront CDN jakelemassa liikennettä globaalisti.

### 2. Taustajärjestelmä ja API
- **Framework:** FastAPI Pythonilla
- **Hosting:** AWS Lambda + API Gateway
- **Tehtävä:** Reitittää käyttöliittymän CRUD-pyynnöt ja ohjata tekoälyn analyysipyynnöt SQS-jonoon asynkronisesti.

### 3. Tietokannat
- **Relaatiotietokanta (Vuorovaikutus ja portfoliot):** Amazon Aurora Serverless v2 (PostgreSQL). 
  - Erityistä: Kytketty **Data API**:lla, jolloin sitä voidaan kutsua suoraan HTTPS-pyynnöillä Lambdasta. Tämä poistaa täysin VPC:stä (Virtual Private Cloud) johtuvat verkottumisongelmat ja pakotukset.
- **Vektoritietokanta (Dokumentit ja uutiset):** Kustannustehokas "S3 Vectors". 
  - Markkinatutkimukset ja artikkelit muunnetaan vektoreiksi (embeddings) ja tallennetaan S3-ämpäriin JSON-tiedostoina. Upotusten (embeddings) luontiin käytetään omaa Amazon SageMaker Serverless -päätepistettä (malli: `all-MiniLM-L6-v2`). Kustannus voi olla jopa 90 % halvempi kuin esim. raskailla OpenSearch-klustereilla.

### 4. Tekoälyagentit (LLMs ja MCP)
- **Moduuli:** OpenAI Agents SDK, joka ohjaa Amazon Bedrockin `Nova Pro` (`us.amazon.nova-pro-v1:0`) -kielimallia LiteLLM-rajapinnan läpi.
- **Moniagenttiorganisaatio (Agent Orchestra):** Asynkroninen tapahtumapohjainen arkkitehtuuri, jossa Amazon **SQS (Simple Queue Service)** jakaa työt.
  1. **Planner (Orkestraattori):** Ottaa vastaan SQS-jonolta käyttäjän pyynnön ja jakaa sen rinnakkain useille ali-agenteille. 
  2. **Tagger:** Luokittelee instrumentteja.
  3. **Reporter:** Kirjoittaa syvällisiä salkkuanalyysejä ja sijoitusraportteja.
  4. **Charter:** Generoi visualisointidataa kaavioita varten.
  5. **Retirement:** Ennustaa salkun kehitystä ja laskee eläkeennusteita.
  - Kaikki rinnakkaiset ali-agentit ovat itsenäisiä AWS Lambda -funktioita, ja ne kirjoittavat palaavan datansa suoraan tietokannan `jobs`-taulun JSONB-kenttiin.

### 5. Itsenäinen Researcher-mikropalvelu
- **Hosting:** AWS App Runner (jatkuvasti pyörivä ja automaattisesti skaalautuva Docker-kontti)
- **Tehtävä:** Tutkii internetiä itsenäisesti tekoälyn voimin ja syöttää datan Ingest-Lambdan kautta vektoritietokantaan S3:een.
- **Tekijä:** Nova Pro + **Playwright MCP Server**, joka mahdollistaa web-sivujen lukemisen.
- **Ajoitus:** Laukaistaan Amazon EventBridge Schedulerilla automaattisesti esimerkiksi kahden tunnin välein jatkuvan uutisvirran ja datan keräämiseksi.

### 6. Enterprise-tason ominaisuudet
- **Tarkkailu (Observability):** LangFuse SDK tekoälyagenttien promptien, vastausten, työkalukutsujen ja viiveiden jäljittämiseen (tracing).
- **Infrastruktuurin valvonta:** Amazon CloudWatch -dashboardit ja automaattiset hälytykset (Alarms), jos jokin agenteista alkaa kaatuilla tai palauttaa virheitä.
- **Tietoturva:** AWS WAF API:n edessä, sekä GuardDuty uhkien havainnointiin.

---

## 👣 Kuinka imitoida projektin vaiheet myöhemmin? (Step-by-step Pystytys)

Kun haluat luoda samantyylisen infra- tai ohjelmistokokonaisuuden alusta alkaen, suorita deployment tässä järjestyksessä. **Jokainen Terraform-deploy-kansion osa on tässä projektissa täysin erillinen**, ilman keskitettyä Terraform-tilaa (state). Tämän ansiosta järjestelmän rakentaminen (ja tuhoaminen kuluja säästäessä) on erittäin oppimisystävällistä modulaarisesti vaihe kerrallaan.

**HUOM:** Ennen pystytystä varmista AWS CLI -autentikaatio, ja kopioi aina edellisen askeleen tärkeimmät AWS ARN:t ja API-avaimet juuressa olevaan `.env`-tiedostoon sekä seuraavan vaiheen `terraform.tfvars`-tiedostoon!

### 1. Perusteet ja upottamismallit (Guides 1 & 2)
1. **IAM-oikeudet:** Luo AWS:ssä riittävät IAM-oikeudet kehittäjäkäyttäjällesi (esim. `AlexAccess` ryhmä Custom policyillä). Pyydä Amazon Bedrockista (us-west-2/us-east-1) myös käyttöoikeus malleihin (Nova Pro, OpenAI OSS).
2. **SageMaker:** Mene `terraform/2_sagemaker` ja aja `terraform apply`. Tämä pystyttää SageMaker Serverless Endpointin `all-MiniLM-L6-v2` -embeddaavalle mallille.

### 2. Vektoritietokanta Ingest (Guide 3)
1. **Vector DB (S3 Vectors):** Mene `terraform/3_ingestion` ja pystytä API Gateway, S3-vektoriämpäri ja Ingest Lambda.
2. Nyt sinulla on rajapinta asiakirjojen siirtämiseksi talteen vektoritietokantaan. Ota `.env` -tiedostoon talteen API:n päätepiste ja API Key.

### 3. Autonominen Research App (Guide 4)
1. Mene kansioon `terraform/4_researcher`. Aja ensin erikseen pelkän Dockerin säilön (`aws_ecr_repository`) ja IAM-roolin luonti (`terraform apply -target=aws_ecr_repository.researcher` jne.)
2. Aja Python-skripti `uv run deploy.py`. Tämä kääntää App Runneriin suunnatun Docker-kuvan Playwright Chromium -riippuvuuksilla paikallisesti ja siirtää sen ECR-säilöön.
3. Aja loppuun `terraform apply`, joka luo App Runner -instanssin ja EventBridge Schedulerin. 
4. Nyt Alex selaimen avulla kykenee etsimään tietoa netistä reaaliajassa, rikastamaan sen Nova Pro:lla, lähettämään sen Ingest API:in ja S3 Vectoriin semantic searchia varten.

### 4. Relaatiotietokanta ja Portfoliot (Guide 5)
1. Mene `terraform/5_database` ja asenna Aurora Serverless v2 PostgreSQL klusteri, Secrets Manager sekä vaadittavat tietoturvaryhmät.
2. Vie generoidut Cluster ARN ja Secret ARN `.env`-tiedostoosi.
3. Pakene AWS-vaikeuksista ajamalla taulut (`run_migrations.py`) ja ETF-osakkeiden seed datat (`seed_data.py`) heti sisään Data API:n avulla, ilman monimutkaista VPC-tunneleiden asetusten määritystä.

### 5. AI Orkesteri ja Agentit (Guide 6)
1. Suuntaa hakemistoon `terraform/6_agents`. Tämä on sovelluksen ohjelmointimielessä mielenkiintoisin osuus. Aja Terraform, mikä luo kaikki järjestelmän Lambdat (Planner, Tagger, Reporter, Charter, Retirement). 
2. Luonti sisältää myös **Amazon SQS** -viestijonon, josta Orchestrator (Planner) nappaa käyttöliittymän synkroniset pyynnöt, käynnistää asynkronisten ali-agenttien ketjun, ja agentit palauttavat työnsä jäljen Aurora PostgreSQL:ään.

### 6. UI & Turvallisuus (Guides 7 & 8)
1. Määrittele API Gateway + Lambda FastAPI. 
2. Rakenna NextJS ja pystytä se Amazon S3:een (Static Website Hosting) CloudFront-jakelun taakse (`terraform/7_frontend`).
3. Määritä Clerk Authenticationin avaimet käyttöliittymälle, jotta sisäänrakennettu JWT-token autentikaatio pelaa API:n kanssa heti ensimmäisellä klikkauksella.
4. Tuo projekti tuotantovalaistukseen pystyttämällä LangFuse tracing (`LANGFUSE_PUBLIC_KEY` jne. ympäristömuuttujiin) ja asettamalla tuotantotason hälytykset pystyyn (`terraform/8_enterprise`).

---

### Miten kehittää eteenpäin?
Koska järjestelmä noudattaa tarkasti mikropalveluarkkitehtuuria (esim. App Runnerin eristys, Lambdojen vastuiden jako tekoälyssä), voit skaalata järjestelmää täysin modulaarisesti. 
Tulevaisuudessa voit esimerkiksi:
* Lisätä uuden asiantuntija-agentin koodaamalla sille `agent.py` ja `lambda_handler.py` tiedostot ja muuttamalla `terraform/6_agents` reseptejä antamaan se osaksi Planner-orkestraattorin reititystä.
* Laajentaa App Runner / Web Researcher -agentin työkaluja MCP:ssä, sillä Nova Pro kykenee käyttämään kompleksisia työkaluja (vaikka se onkin herkkä "ToolUse invalid sequence" -rajapinnoille ilman retry API -mekanismeja).

---
*Kustannusrakennehälytys:* Älä unohda tuhota varsinkin Auroran PostgreSQL:ää (`terraform destroy` `5_database` hakemistossa) loman/kehitystauon ajaksi, se on tyypillisesti isoin yksittäinen kuluerä (~$1.50+/päivä)!
