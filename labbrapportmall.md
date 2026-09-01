# Labbrapport: praktisk laboration

*Kunskapskontroll 2, IT-säkerhet för utvecklare. Fyll i mallen och lämna in som PDF tillsammans med länken till ditt repo. Riktlängd två till tre sidor.*

**Namn:**
**Datum:**
**Repo (länk till din fork):**
**Applikation som analyserades:**

---

## 1. Kort om applikationen och analysen

Beskriv i några meningar vilken app du analyserade, vad den gör och hur du genomförde analysen. Ange vilka verktyg du använde och hur du körde dem (CodeQL default setup med språk C#, ZAP passiv och aktiv skanning mot vilken adress).

Jag analyserade SakerLabb, ett ärendehanteringssystem. Appen låter kunder skicka in supportärenden som administratörer kan söka i, kommentera och byta status på. Det går att ladda upp bilagor, göra import, diagnostik och hantera administratörer. 

Jag gjorde analysen med både statiskt och dynamisk kodanalys. Den statiska analysen gjorde jag i GitHub med CodeQL och GitHub code scanning med default setup och C# som språk. 
Den dynamiska analysen gjorde jag med OWASP ZAP mot localhost:5080. Jag loggade in i appen och klickade mig runt medan ZAP gjorde en passiv skanning. 

---

## 2. Fem fynd

Fyll i tabellen. Minst ett fynd ska komma från statisk analys (CodeQL) och minst ett från dynamisk analys (ZAP). Spara bevis i form av skärmbild eller rapportutdrag och hänvisa till det per fynd.

| Nr | Källa (CodeQL/ZAP) | Regel-id eller alert | Allvarlighet (+ confidence för ZAP) | Fil och rad eller URL | Verkligt eller falskt positivt | Motivering (2–4 meningar) |
|----|--------------------|----------------------|-------------------------------------|-----------------------|--------------------------------|---------------------------|
| 1 |CodeQL  |cs/web/xss  |High  |SakerLabb.Web/Components/Pages/Tickets.razor:21  |Verkligt  |Skriver ut sökordet oescapat så att skript kan köras i webbläsaren. En angripare kan skicka en länk med skadligt skript som körs hos den som klickar, t.ex. för att stjäla sessionen. Sökordet kommer från användaren och visas rakt av, så fyndet är verkligt.  |
| 2 |CodeQL  |cs/web/cookie-httponly-not-set + cs/web/cookie-secure-not-set  |Medium  |SakerLabb.Web/Services/AuthService.cs:25  |Verkligt  |Inloggningskakan sätts utan flaggorna HttpOnly och Secure. Utan HttpOnly kan skript i webbläsaren läsa kakan, och utan Secure kan den skickas okrypterat. Det gör det lättare för en angripare att komma åt sessionen.  |
| 3 |CodeQL  |cs/unsafe-deserialization-untrusted-input  |Critical  |SakerLabb.Web/Services/ImportService.cs:40  |Verkligt  |Det går att skicka in instruktioner som gör att servern kör koden. Vad som ska göras bestäms i inkommande data istället för i koden. Det betyder att en angripare i praktiken kan ta över servern. Datan tas emot direkt i importen utan kontroll.  |
| 4 |CodeQL  |cs/xml/insecure-dtd-handling  |Critical  |SakerLabb.Web/Services/ImportService.cs:27  |Verkligt  |Appen kan ta emot XML-filer och "lyda" instruktioner. Den borde bara läsa datan. En angripare kan därför få servern att läsa upp lokala filer eller surfa till adresser åt dem. Eftersom XML:en kommer utifrån och läses utan spärr är fyndet verkligt.  |
| 5 |ZAP  |Content Security Policy (CSP) Header Not Set  |Risk: Medium , Confidence: High  |http://localhost:5080/SakerLabb.Web.styles.css  |Verkligt  |Saknar en säkerhetsregel som säger till webbläsaren att endast köra sina egna skript. Det är inget hål i sig, men det tar bort ett skyddsnät och gör XSS-fyndet värre. ZAP bekräftar passivt att headern saknas.  |

Bevis (skärmbilder eller utdrag), numrerade efter fyndet ovan:

**Fynd 1 – cs/web/xss, Tickets.razor:21 (alert #22):**
![Fynd 1: Cross-site scripting i Tickets.razor:21](proof-1-xss.png)

**Fynd 2 – cs/web/cookie-secure-not-set / cookie-httponly-not-set, AuthService.cs:25 (alert #32):**
![Fynd 2: Cookie utan Secure/HttpOnly i AuthService.cs:25](proof-2-cookie.png)

**Fynd 3 – cs/unsafe-deserialization-untrusted-input, ImportService.cs:40 (alert #15):**
![Fynd 3: Osäker deserialisering i ImportService.cs:40](proof-3-deserialization.png)

**Fynd 4 – cs/xml/insecure-dtd-handling, ImportService.cs:27 (alert #20):**
![Fynd 4: Osäker XML/DTD-hantering i ImportService.cs:27](proof-4-dtd.png)

**Fynd 5 – Content Security Policy (CSP) Header Not Set (ZAP, passiv):**
![Fynd 5: CSP-header saknas (ZAP)](proof-5-csp.png)

---

## 3. Prioritering

Rangordna fynden och motivera ordningen med allvarlighetsgrad, exponering och utnyttjbarhet. Vilket tar du först och varför?

Fynd 3 – Allvarlighet: högst av alla. Kan ge full kontroll över servern (kodkörning). Exponering: ligger i importfunktionen som tar emot data utifrån. Utnyttjbarhet: en angripare behöver bara skicka in preparerad JSON.

Fynd 4 – Allvarlighet: hög, men konsekvensen är främst filläsning och utgående anrop (SSRF), inte direkt kodkörning. Exponering: samma importväg, tar emot XML utifrån. Utnyttjbarhet: medel, kräver en preparerad XML-fil med extern entitet.

Fynd 1 – Allvarlighet: hög, men drabbar klienten och inte servern. Exponering: publik sökfunktion som är lätt att nå. Utnyttjbarhet: hög, men kräver att ett offer klickar på en preparerad länk.

Fynd 2 – Allvarlighet: medel. Gör det lättare att komma åt sessionen men är ingen egen väg in. Exponering: gäller inloggningskakan för alla användare. Utnyttjbarhet: kräver oftast ett annat fel först, t.ex. XSS eller okrypterad trafik.

Fynd 5 – Allvarlighet: lägst. Inget hål i sig utan ett saknat skyddslager. Exponering: gäller hela appen. Utnyttjbarhet: kan inte utnyttjas i sig men förvärrar XSS-fyndet.


Jag tar Fynd 3 först, eftersom det har den värsta konsekvensen (full kontroll över servern) och den lägsta tröskeln att utnyttja. Det räcker att skicka in preparerad JSON till importen.

---

## 4. Åtgärder (minst tre)

Använd mönstret nedan per åtgärdat fynd. Varje åtgärd ska gå att spåra tillbaka till ett fynd i tabellen ovan, och beviset efter ska vara en **ny körning av verktyget**, inte din egen kod.

### Åtgärd 1

```
Fynd: 3 - cs/unsafe-deserialization-untrusted-input     (nr och regel-id/alert från tabellen ovan)
Plats: SakerLabb.Web/Services/ImportService.cs:40      (fil och rad, eller URL)
Bevis före:  (skärmbild eller rapportutdrag som visar fyndet)
Bedömning:   (verkligt eller falskt positivt, kort motiverat)
Åtgärd:      Ta bort TypeNameHandling.All. En rad
Bevis efter: (ny körning: CodeQL-alerten står som Fixed, eller ZAP-larmet är borta ur den nya rapporten)
```

### Åtgärd 2

```
Fynd: 4 - cs/xml/insecure-dtd-handling
Plats: SakerLabb.Web/Services/ImportService.cs:27
Bevis före:
Bedömning:
Åtgärd: DtdProcessing.Prohibit + XmlResolver = null.
Bevis efter:
```

### Åtgärd 3

```
Fynd: 1 - cs/web/xss
Plats: SakerLabb.Web/Components/Pages/Tickets.razor:21
Bevis före:
Bedömning:
Åtgärd: Ta bort MarkupString, skriv @Search.
Bevis efter:
```

---

## 5. Eventuella bortval

Om du valt att inte åtgärda ett fynd, skriv ned tre saker per bortval: risken, motivet och den kompenserande kontrollen. Sätt gärna ett datum för omprövning.

*Skriv här, eller skriv "inga bortval".*
