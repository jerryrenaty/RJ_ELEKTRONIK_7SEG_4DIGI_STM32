# Pilote Afficheur 7 Segments 4 Digits (Non-Bloquant) pour STM32

Ce composant logiciel est un pilote (driver) en langage C optimisé pour microcontrôleurs STM32 (via la bibliothèque HAL). Il permet de piloter un afficheur à 7 segments de 4 chiffres (digits) en utilisant le multiplexage temporel géré par interruption matérielle (Timer), évitant ainsi tout blocage du processeur (`HAL_Delay`).

## 🚀 Caractéristiques

- **Non-bloquant** : L'affichage se rafraîchit en arrière-plan via une interruption de Timer.
- **Polyvalent** : Supporte nativement les afficheurs à **Anode Commune** et **Cathode Commune**.
- **Simplifié** : Configuration matérielle basée directement sur les `#define` générés par STM32CubeMX.
- **Sécurisé** : Gestion automatique des signes négatifs, du dépassement de capacité (`----`) et des points décimaux.

---

## 🛠️ Configuration Matérielle (CubeMX)

### 1. Assignation des Broches (GPIO)
Les broches doivent être configurées en mode **GPIO_Output** (Vitesse : *Low* ou *Medium*) dans l'outil graphique CubeMX avec les labels utilisateur exacts suivants :


| Segment / Digit | Label CubeMX | Exemple de Broche Usée |
| :--- | :--- | :--- |
| **Segment A** | `A_Pin` / `A_GPIO_Port` | GPIOD, Pin 13 |
| **Segment B** | `B_Pin` / `B_GPIO_Port` | GPIOD, Pin 12 |
| **Segment C** | `C_Pin` / `C_GPIO_Port` | GPIOD, Pin 11 |
| **Segment D** | `D_Pin` / `D_GPIO_Port` | GPIOD, Pin 10 |
| **Segment E** | `E_Pin` / `E_GPIO_Port` | GPIOD, Pin 9 |
| **Segment F** | `F_Pin` / `F_GPIO_Port` | GPIOD, Pin 8 |
| **Segment G** | `G_Pin` / `G_GPIO_Port` | GPIOB, Pin 15 |
| **Digit 1 (Gauche)** | `D1_Pin` / `D1_GPIO_Port` | GPIOC, Pin 8 |
| **Digit 2** | `D2_Pin` / `D2_GPIO_Port` | GPIOC, Pin 7 |
| **Digit 3** | `D3_Pin` / `D3_GPIO_Port` | GPIOC, Pin 6 |
| **Digit 4 (Droite)** | `D4_Pin` / `D4_GPIO_Port` | GPIOD, Pin 15 |

### 2. Configuration du Timer (TIMx)
Pour un affichage fluide sans scintillement visible pour l'œil humain :
1. Activez un timer (ex: `TIM3`) dans CubeMX.
2. Réglez sa fréquence d'interruption pour obtenir **~400 Hz** (ce qui équivaut à 100 Hz par chiffre).
   - *Exemple de calcul pour une horloge système à 72 MHz* : 
     - Prescaler (PSC) = `71` (Horloge divisée par 72 = 1 MHz)
     - Counter Period (ARR) = `2499` (1 MHz / 2500 = 400 Hz -> Interruption toutes les **2.5 ms**).
3. Dans l'onglet **NVIC Settings** du Timer, cochez la case **Enabled** pour activer l'interruption globale.

---

## 💻 Intégration du Code

### 1. Fichiers du projet
Ajoutez `RJ_ELEKTRONIK_7SEG_4DIGI_STM32.h` dans votre dossier d'en-têtes (`Core/Inc`) et `RJ_ELEKTRONIK_7SEG_4DIGI_STM32.c` dans votre dossier des sources (`Core/Src`).

### 2. Liaison de l'interruption (dans `main.c`)
Ajoutez la fonction de rappel (Callback) de HAL pour lier le rafraîchissement à l'événement du Timer.

```c
/* USER CODE BEGIN 4 */
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance == TIM3) { // Changez TIM3 par votre instance choisie
        SevenSeg_Mux_Callback();
    }
}
/* USER CODE END 4 */
```

### 3. Exemple complet d'utilisation (dans `main.c`)

```c
#include "RJ_ELEKTRONIK_7SEG_4DIGI_STM32.h"

int main(void) {
    // Initialisations automatiques STM32CubeMX
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_TIM3_Init();

    // 1. Initialisation simplifiée du driver (COMMON_CATHODE ou COMMON_ANODE)
    SevenSeg_Init(COMMON_CATHODE);

    // 2. Démarrage du Timer en mode Interruption
    HAL_TIM_Base_Start_IT(&htim3);

    int16_t mon_compteur = -50;

    while (1) {
        // Envoie le nombre au tampon mémoire (Mise à jour instantanée et non-bloquante)
        // Paramètres : (Nombre, Activer_Point_Décimal, Position_Point_0_à_3)
        SevenSeg_UpdateNumber(mon_compteur, true, 2);

        mon_compteur++;
        if (mon_compteur > 1500) {
            mon_compteur = -50;
        }

        // L'afficheur reste parfaitement fluide pendant ce délai !
        HAL_Delay(100); 
    }
}
```

---

## 📝 Fonctions Disponibles

- `void SevenSeg_Init(DisplayType type);`  
  Initialise le type d'affichage (`COMMON_CATHODE` ou `COMMON_ANODE`).
- `void SevenSeg_UpdateNumber(int16_t number, bool point_on, uint8_t point_pos);`  
  Met à jour le nombre affiché en mémoire (Plage supportée : -999 à 9999).
- `void SevenSeg_Mux_Callback(void);`  
  Fonction interne de traitement du multiplexage (à placer dans l'interruption du Timer).
- `void SevenSeg_Clear(void);`  
  Éteint complètement tous les segments de l'afficheur.

---
© 2026 RJ Elektronik - Tous droits réservés.
