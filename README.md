# Pilote Afficheur 7 Segments 4 Digits (Non-Bloquant) pour STM32

[![Platform: STM32](https://shields.io)](https://st.com)
[![Framework: STM32CubeHAL](https://shields.io)](https://st.com)
[![Language: C](https://shields.io)](https://wikipedia.org)
[![License: MIT](https://shields.io)](https://opensource.org)

Ce composant logiciel est un pilote (*driver*) léger écrit en langage C et optimisé pour les microcontrôleurs STM32 via la bibliothèque HAL. Il permet de piloter un afficheur à 7 segments de 4 chiffres (digits) en utilisant un multiplexage temporel géré de manière asynchrone par interruption matérielle (Timer), évitant ainsi tout blocage ou ralentissement du processeur (`HAL_Delay`).

---

## 🚀 Caractéristiques

* **Non-bloquant** : L'affichage se rafraîchit entièrement en arrière-plan via une interruption de Timer.
* **Polyvalent** : Supporte nativement les afficheurs à **Anode Commune** et **Cathode Commune**.
* **Simplifié** : Configuration matérielle basée directement sur les `#define` générés par STM32CubeMX.
* **Sécurisé** : Gestion automatique des signes négatifs, du dépassement de capacité (`----`) et des points décimaux.
* **Autonome** : L'interruption est captée directement par le driver grâce à l'enregistrement du pointeur de Timer lors de l'initialisation.

---

## 🛠️ Configuration Matérielle (CubeMX)

### 1. Assignation des Broches (GPIO)
Configurez vos broches en mode **GPIO_Output** (Vitesse : *Low* ou *Medium*) dans l'outil graphique CubeMX, puis renommez-les avec les labels utilisateur exacts suivants :


| Segment / Digit | Label CubeMX Requis | Exemple de Broche Réelle |
| :--- | :--- | :--- |
| **Segment A à G** | `A_Pin` à `G_Pin` | `PA1` à `PA7` |
| **Point Décimal** | `DP_Pin` | `PB7` |
| **Digit 1 (Gauche)**| `D1_Pin` | `PB6` |
| **Digit 2** | `D2_Pin` | `PB5` |
| **Digit 3** | `D3_Pin` | `PB4` |
| **Digit 4 (Droite)**| `D4_Pin` | `PB3` |

### 2. Configuration du Timer (TIMx)
Pour obtenir un affichage parfaitement fluide et sans aucun scintillement visible pour l'œil humain :
1. Activez un timer (par exemple `TIM3`) dans CubeMX.
2. Configurez ses registres pour obtenir une fréquence d'interruption globale de **~400 Hz** (soit 100 Hz stables par chiffre).
   * *Exemple de calcul pour une horloge système cadencée à **96 MHz** :*
     * **Prescaler (PSC)** : `239` (Horloge divisée par 240 = 400 kHz)
     * **Counter Period (ARR)** : `999` (400 kHz / 1000 = 400 Hz $\rightarrow$ Une interruption toutes les **2.5 ms**).
3. Dans l'onglet **NVIC Settings** du Timer choisi, cochez impérativement la case **Enabled** pour activer l'interruption globale.

---

## 💻 Intégration du Code

### 1. Fichiers du projet
* Copiez le fichier `RJ_ELEKTRONIK_7SEG_4DIGI_STM32.h` dans le dossier de vos en-têtes (`Core/Inc/`).
* Copiez le fichier `RJ_ELEKTRONIK_7SEG_4DIGI_STM32.c` dans le dossier de vos sources (`Core/Src/`).

### 2. Gestion automatique de l'interruption
Le driver intercepte automatiquement la routine d'interruption globale grâce au mécanisme des fonctions de rappel (*callbacks*) de la HAL. Le code ci-dessous est déjà inclus dans le fichier `.c` :

```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (phtim != NULL && htim->Instance == phtim->Instance) { 
        SevenSeg_Mux_Callback();
    }
}
```

### 3. Exemple complet d'implémentation (`main.c`)

```c
#include "main.h"
#include "RJ_ELEKTRONIK_7SEG_4DIGI_STM32.h"

/* Variables privées générées par CubeMX */
TIM_HandleTypeDef htim3; 

int main(void) {
    /* Initialisations du système STM32 */
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_TIM3_Init();

    /* 1. Initialisation du driver avec le type et le pointeur du Timer */
    SevenSeg_Init(COMMON_CATHODE, &htim3);

    /* 2. Démarrage matériel du Timer en mode Interruption */
    HAL_TIM_Base_Start_IT(&htim3);

    int16_t mon_compteur = -50;

    while (1) {
        /* Met à jour le nombre en mémoire tampon (Instantané et non-bloquant) */
        /* Arguments : (Nombre, Activer_Point, Position_du_Point_0_à_3) */
        SevenSeg_UpdateNumber(mon_compteur, true, 2);

        mon_compteur++;
        if (mon_compteur > 1500) {
            mon_compteur = -50;
        }

        /* L'affichage reste net et fluide en arrière-plan pendant ce délai ! */
        HAL_Delay(100); 
    }
}
```

---

## 📝 API : Fonctions Disponibles

### `void SevenSeg_Init(DisplayType type, TIM_HandleTypeDef *htim);`
Initialise la structure de l'afficheur (`COMMON_CATHODE` ou `COMMON_ANODE`) et enregistre dynamiquement l'adresse mémoire de l'instance du matériel pour automatiser le multiplexage sous interruption.

### `void SevenSeg_UpdateNumber(int16_t number, bool point_on, uint8_t point_pos);`
Met à jour le tampon de données numériques de l'afficheur. 
* **Plage de valeurs supportée** : `-999` à `9999`. 
* Si la valeur dépasse `9999`, l'écran affiche un message de débordement sécurisé (`----`).

### `void SevenSeg_Clear(void);`
Éteint immédiatement l'intégralité des 4 chiffres et nettoie la mémoire tampon associée.

---
## 📄 Licence
Ce projet est sous licence MIT. Consultez le fichier `LICENSE` pour plus de détails.

© 2026 **RJ Elektronik** - Tous droits réservés.
