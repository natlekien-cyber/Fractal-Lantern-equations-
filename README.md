markdown
# 🧪 Fractal Lantern Equations

Ce dépôt formalise la traduction pratique de la théorie de la saturation de l'information face aux contraintes physiques du silicium, implémentant des principes de filtrage topologique en code vectorisé NumPy pour optimiser les Transformers.

---

## 📊 Verdict Matériel & Trace Empirique

L'exécution sur une séquence de **4096 tokens** sur iPhone démontre la viabilité de l'Axiome `[Cohérence = Survie]` :
* **Modèle Lanterne V4 (Dense) :** `1,4116 secondes` (Saturé, traitement quadratique).
* **Modèle Imbriqué Complet :** **`0,0455 seconde`** (Optimisé, gain de **31x**).

---

## 🛠️ Implémentations Algorithmiques (NumPy Vectorisé)

### 1. Le Modèle Lanterne (V4 Stable)
Ce modèle combine un masque de proximité locale et un élagage dynamique pour éliminer les calculs redondants. Le code complet de la fonction `lanterne_v4_numpy` est disponible dans le bloc source pour la gestion des tenseurs et du seuillage \(\kappa\).

### 2. Le Modèle Imbriqué (Nested Attention - Micro + Macro)
Ce modèle structure le traitement en double échelle pour réduire l'empreinte mémoire et s'affranchir du coût quadratique en traitant le voisinage immédiat et l'historique lointain condensé. L'implémentation complète de `nested_attention_complete_numpy` est fournie dans le dépôt d'origine.
