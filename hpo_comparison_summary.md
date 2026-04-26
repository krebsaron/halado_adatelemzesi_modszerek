# DDoS Detekciós Modellek Összehasonlítása: HPO, Generalizáció és Adatkiegyenlítés

Ez a dokumentum összefoglalja a `SCLDDoS2024_SetA` és `SetB` adathalmazokon futtatott modellek összehasonlító elemzését. A vizsgálat célja a gépi tanulási modellek (Logistic Regression, Random Forest, Gradient Boosting) teljesítményének, a hiperparaméter-hangolás (HPO) hatásának, valamint az adatkiegyenlítési stratégiáknak (SMOTE és beépített class weight) a kiértékelése volt.

---

## 1. Módszertan

- **Kísérleti Elrendezés**: 
  - **Betanítás és Validáció**: Kizárólag a `SetA` adathalmaz train és validation (10%) split-jein történt (a data leakage elkerülése végett).
  - **Kiértékelés (Tesztelés)**: 
    - `SetA test` split (20%): A modell in-distribution (ismert hálózat/eloszlás) teljesítményének mérésére.
    - `SetB external`: A modell out-of-distribution (ismeretlen hálózati minta, új struktúra) teljesítményének és a **generalizálódási képességnek** a mérésére - ami a legfontosabb a prediktív védelemben.
- **Szcenáriók (Scenarios)**:
  - `regular_class_weight`: Algoritmusba épített `class_weight="balanced"` használata a felülreprezentált osztályok lefojtására.
  - `smote_no_class_weight`: Előzetes szinkretikus adatgenerálás (SMOTE) a tréning set kisebbségi osztályain, a betanítási súlyok kikapcsolásával.
- **Futtatások**:
  - **Base (Alap)**: Alapértelmezett, fix robusztus hiperparaméterekkel (pl. RF n_estimators=350).
  - **Tuned (HPO)**: Grid search (vagy random search) alapú hiperparaméter-optimalizáció `f1_macro` metrikára a SetA validation adatokon.

---

## 2. Fő Megfigyelések és Következtetések (Prezentációhoz)

### A) Melyik modell teljesít a legjobban? (Base vs. Tuned Eredmények)
A **Gradient Boosting (HistGB/XGBoost/LightGBM)** konzisztensen a legjobb eredményt adta, míg a Random Forest szorosan követte, de hajlamosabb volt az overfitting-re. A Logistic Regression a komplex, nemlineáris összefüggések miatt teljesen alulteljesített.

- **SetA F1 (In-Distribution)**: A Tuningolt Boosting és a Tuned Random Forest is ~0.84 - 0.85 F1-Score-t értek el.  
- **SetB F1 (Generalizáció)**: A Tuningolt Random Forest jelentős (~0.51) eredményeket mutatott, felülmúlva (vagy megközelítve mélyebben) a Boosting SetB (0.50) generalizációját a Regular Class Weight esetében. Alapvetően egyiken sem tapasztaltak >0.60-as erős F1 SetB skálát, jelezve, hogy a DDoS forgalom mintázatai drasztikusan eltérnek SetA és SetB köztük.

### B) Hiperparaméter-Hangolás (HPO) Hatása: Generalizáció vs. Overfit
**Tapasztalat**: A HPO csak minimálisan javított az in-distribution (SetA) teljesítményen, azonban rávilágított egy súlyos problémára a Random Forest esetében: az overfittingre.

1. **Overfitting Növekedése**: A Tuningolt modellek jobban optimalizáltak a SetA-ra, amitől a SetA-SetB "generalizációs rés" (vagy overfitting drop) enyhén még **nőtt is**. 
2. **Kisebb előnyök rejtett paraméterekkel**: A HPO megtalálta azt, hogy Boosting esetén a magasabb learing rate (0.1) és több fa (400) stabilabb macro f1-et ad test halmazon. Ellenben ez SetB-n (external) a random forest paraméterek visszaszorítása (max_depth=22) miatt minimális external javulást tett.
3. **Végső következtetés**: A HPO a jelen setupban **nem oldotta meg** az adathalmaz-váltásból adódó durva "domain shift" problémát. SetB-n drasztikusan esnek a komplex modellek (SetA F1 ~0.85 -> SetB F1 ~0.50). 

### C) Adatkiegyenlítés: Class Weight vs. SMOTE
A nagyon unbalanced adat miatt kritikusan fontos.
- **In-distribution (SetA-n)**: A SMOTE rendre kicsivel gyengébben szerepelt kiegyensúlyozott pontosságban, és több illesztési időt vett el.
- **Out-of-distribution (SetB-n)**: Itt egy nagyon érdekes eset látható. Az `smote_no_class_weight` **Boosting** modell a SetB F1-ben az alapmodellnél **jobban teljesített**, ellenben a **Random Forestnél lerontotta a generalizációt** (0.51 -> 0.45 F1).
- **Megállapítás**: Biztonsági szempontból a beépített `class_weight="balanced"` használata a legmegbízhatóbb (regular_class_weight scenario). Sokkal gyorsabb, kevésbé viszi be a modellt "synthetic" túltanulásba a tréning set-en ("feature leak" nélkül adaptál), és a valós teszteken (SetB) a Forest és LogReg esetén lényegesen jobb volt.

---

## 3. Összegzés és Javaslatok

1. **Fa-alapú modellek használata**: A Gradient Boosting implementáció bizonyult a legrobusztusabbnak (gyors, skálázható, F1=~0.85).
2. **Overfitting és Domain Shift**: A hatalmas degradáció a SetB-n azt mutatja, hogy az in-domain tuning (HPO SetA validáción) csak ront a generalizáláson. Javasolt a prediktív modellt vegyes hálózati topológián (pl. Stratified GroupKFold SetA+SetB) képezni a jövőben.
3. **Példátlanul egyszerű balanszálás**: Termelési környezetbe a `regular_class_weight` stratégiát javasoljuk algoritmus szinten kezelve, mert a SMOTE többszörös betanítási idővel generál rosszabb out-of-distribution predikciókat faközpontú architektúrák esetén.