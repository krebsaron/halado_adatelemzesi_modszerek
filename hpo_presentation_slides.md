# DDoS Detekció: Modellek Értékelése és HPO Hatása (Prezentáció)

## 1. Dia: Címoldal és Célkitűzés

**A dia tartalma:**
* **Téma:** DDoS detektáló gépi tanulási modellek teljesítményének összehasonlítása.
* **Cél:** Megvizsgálni a hiperparaméter-hangolás (HPO) hatását, az overfitting mértékét és az adatkiegyenlítési stratégiákat (SMOTE vs. Class Weight).
* **Fő kérdés:** Melyik beállítás védi legjobban a rendszert egy eddig ismeretlen (out-of-distribution) támadási mintával szemben?

**Mely plotokat rakjam be:**
* (Ide nem feltétlenül kell plot, esetleg egy illusztráció a DDoS forgalomról vagy egy egyszerű pipeline ábra.)

**Prezentációs jegyzet (amit mondhatsz):**
> "Üdvözlök mindenkit! Ebben a prezentációban a DDoS támadásokat felismerő modelljeink kiértékelését mutatom be. Nem csak azt vizsgáltuk meg, hogy egy adott - már ismert - hálózati forgalmon mennyire pontosan detektálnak a modellek, hanem azt is, hogy mit reagálnak egy teljesen ismeretlen hálózati forgalomra. Megnéztük, hogy a hiperparaméter-hangolás javít-e ezen, illetve melyik adatkiegyenlítési technika a legstabilabb éles körülmények között."

---

## 2. Dia: Kísérleti Módszertan

**A dia tartalma:**
* **Modellek:** Logistic Regression, Random Forest, Gradient Boosting (XGBoost/LightGBM).
* **Két fő adathalmaz:**
  * **SetA:** In-distribution (A modell ezen tanul és validál, a SetA test része az ismert mintákat méri).
  * **SetB:** Out-of-distribution / External (Ismeretlen minta, a valódi generalizációs képességet méri).
* **Vizsgált dimenziók:**
  * Alapmodell (Base) vs. Hangolt (Tuned/HPO)
  * SMOTE (Szintetikus adatgenerálás) vs. Regular Class Weight.

**Mely plotokat rakjam be:**
* (Ide sem kell feltétlenül plot, esetleg egy táblázat a modellekről és a SetA-SetB szeparációról.)

**Prezentációs jegyzet (amit mondhatsz):**
> "A teszteléshez három eltérő komplexitású modellt választottunk: Logisztikus Regressziót a baseline miatt, valamint Random Forest és Gradient Boosting algoritmusokat. A legfontosabb a kiértékelés módja: a modellt a SetA adathalmazon tanítottuk be, így a 'SetA teszt' eredménye azt mutatja, mennyire ismeri fel azt, amit már látott hasonlót. A 'SetB' viszont egy teljesen más eloszlás, ez reprezentálja a 'valóságot', egy váratlan támadást. Kíváncsiak voltunk, mit tesz a HPO és a SMOTE az eredményekkel."

---

## 3. Dia: Teljesítmény és Generalizációs probléma

**A dia tartalma:**
* **SetA (Ismert minta):** Kiváló eredmények, a komplex modellek (Boosting, Forest) F1 Macro értéke stabilan magas (~0.85).
* **SetB (Ismeretlen minta):** Hatalmas teljesítményesés, az F1 pontszám drasztikusan 0.50 környékére zuhan ("Domain shift").
* **Logisztikus Regresszió:** Jelentősen alulteljesíti a fa-alapú algoritmusokat minden helyzetben.

**Mely plotokat rakjam be:**
* **"F1 Macro összehasonlítás: Alap vs Hangolt (Adathalmazok szerint)"** (A notebook első catplot-ja, ahol látszik a két dataset közötti hatalmas oszlopmagasság-különbség).

**Prezentációs jegyzet (amit mondhatsz):**
> "Ahogy az ábrán is látható, a SetA teszthalmazon a fa-alapú modellek – különösen a Gradient Boosting – kiválóan, 85 százalékos F1 pontosság felett teljesítenek. Viszont, amikor ráeresztjük őket a SetB-re, vagyis az external adatokra, a teljesítmény 50 százalék környékére zuhan. Ez egyértelműen az úgynevezett 'domain shift' jelensége: a támadási minták jelentősen eltérnek a két hálózatban. A Logisztikus Regresszióról hamar kiderült, hogy nem tudja leképezni ezt a komplexitást, így a továbbiakban a Boosting és Forest modellekre fókuszálunk."

---

## 4. Dia: A Hiperparaméter-hangolás (HPO) Ára: Overfitting

**A dia tartalma:**
* A HPO (Hiperparaméter tuning) csak a validációs halmazon (SetA) optimalizál.
* **Tapasztalat:** A Tuning növelte az overfittinget a betanító halmazra!
* **A generalizációs rés (veszteség):** A hangolt modellek esetében a SetA és SetB közötti különbség megnőtt.

**Mely plotokat rakjam be:**
* **"Overfitting: F1 Macro esés mértéke SetA-ról SetB-re"** (A notebook második plot-ja, bárdiagram). Ezen jól látszik az esés mértéke a Base vs. Tuned között!

**Prezentációs jegyzet (amit mondhatsz):**
> "Következőként azt vizsgáltuk, segít-e a paraméterek hangolása. Az ábrán az 'Overfitting' mértéke látható: hogy mennyit esett az F1 pontszám a SetA-ról a SetB-re váltáskor. Nagyon érdekes megfigyelés, hogy a HPO – bár a validációs halmazon javulást mutatott – éles (SetB) környezetben sok esetben rontott a helyzeten! Mivel a HPO a SetA mintázataihoz láncolta jobban a modellt, az egyre specifikusabb szabályokat tanult meg. Vagyis a tuning itt túltanulást, overfittinget eredményezett."

---

## 5. Dia: Adatkiegyenlítés (SMOTE vs. Class Weight)

**A dia tartalma:**
* **SMOTE hátrányai:** Különösen a fa-alapú modelleknél (pl. Random Forest) rontotta az ismeretlen adatok felismerését. Hajlamosabb szintetikus szabályok ("memórizálás") betanulására.
* **Class Weight (Beépített súlyozás):** Gyorsabb módszer, nem generál szintetikus adatot, és SetB-n jobban generalizál. A legstabilabb választás.

**Mely plotokat rakjam be:**
* **"SMOTE vs Class Weight Hatása F1 Macro-ra (Base vs Tuned)"** (A notebook harmadik catplot-ja).

**Prezentációs jegyzet (amit mondhatsz):**
> "Mivel az osztályok nagyon kiegyensúlyozatlanok, vizsgáltuk a SMOTE illetve az algoritmusba beépített sima osztálysúlyozás (class weight) teljesítményét. A SMOTE a tréning adatokon küzd a kisebbségi osztályokért, de az ábrán jól látszik a hatása a generalizációra: a Random Forest esetében például a SMOTE drasztikusan, közel 5-6 százalékkal lerontotta a SetB-n mért F1 macro eredményt a sima Class Weighthez képest. Éles, robusztus rendszerekben a beépített class weight a nyerő, mert nem viszi be az algoritmust a 'szintetikus' adatok betanulásának csapdájába és sokkal gyorsabb is."

---

## 6. Dia: Összegzés és Élesítés (Actionable Insights)

**A dia tartalma:**
* **A legjobb algoritmus:** Gradient Boosting (XGBoost / LightGBM) adta a legstabilabb, legjobb eredményt.
* **Adatkiegyenlítési stratégia:** Termelési (Production) környezetben a robusztus `class_weight="balanced"` preferálandó a SMOTE-tal szemben.
* **Okulás az Overfittingből:** Szigorú, SetA-specifikus hiperparaméter-tuning ker kerülendő, ha a valós forgalom nagyon diverz. Kiterjedtebb tréning stratégiára van szükség (pl. SetA és SetB keverése).

**Mely plotokat rakjam be:**
* (Opcionális: egy mátrix/táblázat a végső legjobb SetA és SetB metrikákkal, esetleg a legjobb Confusion Matrix a SetB-ről).

**Prezentációs jegyzet (amit mondhatsz):**
> "Összefoglalva: a jövőbeli feladatainkhoz és élesítéshez a Gradient Boosting technológiát javasoljuk, mivel ez biztosította folyamatosan a legmagasabb F1 pontszámokat. Számítási és megbízhatósági okokból elvetjük a SMOTE-ot, helyette a beépített osztálysúlyozást fogjuk alkalmazni. Végül, a generalizációs probléma miatt a jövőben egy jóval diverzebb adathalmazzal kell megtanítanunk a rendszert – akár SetA és SetB keverékével –, és kerülnünk kell az egyetlen specifikus hálózati topológiára kihegyezett agresszív paraméter-optimalizációt. Köszönöm a figyelmet!"