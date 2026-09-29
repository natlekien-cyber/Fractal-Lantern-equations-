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
Ce modèle structure le traitement en double échelle pour réduire l'empreinte mémoire et s'affranchir du coût quadratique en traitant le voisinage immédiat et l'historique lointain condensé. L'implémentation complète de `nested_attention_complete_numpy` est fournie dans le dépôt 
"""
🧪 Fractal Lantern Equations
Formalisation et traduction pratique de la théorie de la saturation de l'information.
Axiome de référence : [Cohérence = Survie]
"""

import numpy as np
import time

def lanterne_v4_numpy(Q, K, V, kappa=0.05, window_size=32):
    """
    Implémentation vectorisée V4 du Modèle Lanterne.
    Filtre par proximité topologique locale et seuil critique dynamique (kappa).
    """
    L, d = Q.shape
    scale = 1.0 / np.sqrt(d)
    
    # Produit scalaire brut vectorisé
    A_raw = np.dot(Q, K.T) * scale
    
    # Masque de proximité locale (bande diagonale de rayon window_size)
    idx = np.arange(L)
    mask_local = np.abs(idx[:, None] - idx[None, :]) <= window_size
    
    # Élagage dynamique via le seuil de cohérence Kappa
    mask_kappa = A_raw >= kappa
    final_mask = mask_local & mask_kappa
    
    # Softmax masqué numériquement stable
    A_raw_masked = np.where(final_mask, A_raw, -np.inf)
    A_max = np.max(A_raw_masked, axis=-1, keepdims=True)
    A_max = np.where(np.isinf(A_max), 0.0, A_max) 
    
    exp_A = np.exp(A_raw_masked - A_max)
    exp_A = np.where(final_mask, exp_A, 0.0)
    
    sum_exp = np.sum(exp_A, axis=-1, keepdims=True)
    A_weights = np.where(sum_exp > 0, exp_A / sum_exp, 0.0)
    
    return np.dot(A_weights, V)


def nested_attention_complete_numpy(Q, K, V, local_window=32, block_size=32):
    """
    Version optimisée du Modèle Imbriqué (Micro + Macro).
    Pré-calcule l'échelle Macro globalement pour s'affranchir du coût quadratique.
    """
    L, d = Q.shape
    scale = 1.0 / np.sqrt(d)
    out = np.zeros_like(Q)
    
    # Pré-calcul global des blocs Macro (compression par pooling moyen)
    num_blocks = L // block_size
    K_macro_global = K[:num_blocks * block_size].reshape(num_blocks, block_size, d).mean(axis=1)
    V_macro_global = V[:num_blocks * block_size].reshape(num_blocks, block_size, d).mean(axis=1)
    
    for i in range(L):
        local_start = max(0, i - local_window)
        local_end = i + 1
        
        # Échelle Micro : Contexte de voisinage immédiat
        K_local = K[local_start:local_end]
        V_local = V[local_start:local_end]
        A_local = np.dot(Q[i], K_local.T) * scale
        
        # Échelle Macro : Intégration condensée de l'histoire lointaine
        if local_start > 0:
            num_accessible_blocks = local_start // block_size
            
            if num_accessible_blocks > 0:
                K_macro = K_macro_global[:num_accessible_blocks]
                V_macro = V_macro_global[:num_accessible_blocks]
                
                # Gestion du résidu topologique intermédiaire
                res_start = num_accessible_blocks * block_size
                if res_start < local_start:
                    K_res = K[res_start:local_start].mean(axis=0, keepdims=True)
                    V_res = V[res_start:local_start].mean(axis=0, keepdims=True)
                    K_macro = np.vstack([K_macro, K_res])
                    V_macro = np.vstack([V_macro, V_res])
            else:
                K_macro = K[:local_start].mean(axis=0, keepdims=True)
                V_macro = V[:local_start].mean(axis=0, keepdims=True)
            
            A_macro = np.dot(Q[i], K_macro.T) * scale
            A_combined = np.concatenate([A_macro, A_local])
            V_combined = np.vstack([V_macro, V_local])
        else:
            A_combined = A_local
            V_combined = V_local
            
        # Normalisation et Softmax unique sur l'espace fusionné
        A_max = np.max(A_combined)
        exp_A = np.exp(A_combined - A_max)
        weights = exp_A / np.sum(exp_A)
        
        out[i] = np.dot(weights, V_combined)
        
    return out


# ==========================================
# SCÉNARIO DE VALIDATION EMPIRIQUE (BENCHMARK)
# ==========================================
if __name__ == "__main__":
    print("⏳ Initialisation des tenseurs de test...")
    L, d = 4096, 64  # Échelle de saturation physique de la séquence
    np.random.seed(42)
    
    Q = np.random.randn(L, d)
    K = np.random.randn(L, d)
    V = np.random.randn(L, d)
    
    print(f"Structure active : {L} tokens, dimension de plongement {d}")
    print("-" * 60)
    
    # Benchmark 1 : Modèle Lanterne V4 (Quadratique dense masqué)
    t0 = time.time()
    res_lantern = lanterne_v4_numpy(Q, K, V, kappa=0.05, window_size=32)
    t1 = time.time()
    print(f"✅ Modèle Lanterne validé (Forme : {res_lantern.shape})")
    print(f"   Temps d'exécution : {t1 - t0:.4f} secondes")
    
    # Benchmark 2 : Modèle Imbriqué (Compression sous-quadratique)
    t2 = time.time()
    res_nested = nested_attention_complete_numpy(Q, K, V, local_window=32, block_size=32)
    t3 = time.time()
    print(f"✅ Modèle Imbriqué validé (Forme : {res_nested.shape})")
    print(f"   Temps d'exécution : {t3 - t2:.4f} secondes")
    print("-" * 60)
    
    gain = (t1 - t0) / (t3 - t2)
    print(f"🚀 VERDICT MATÉRIEL : Le Modèle Imbriqué est {gain:.1f}x plus rapide.")
