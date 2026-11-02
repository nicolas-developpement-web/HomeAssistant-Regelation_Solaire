# Régulation solaire pour HomeAssistant


## Prompt — Blueprint Home Assistant pour réguler une batterie Izypower Titan en autoconsommation solaire

Agis comme un expert en Home Assistant, YAML, automatisations énergétiques et gestion de batteries domestiques.

Je veux que tu crées un **blueprint Home Assistant complet, fonctionnel, fiable et prêt à importer** pour piloter ma batterie Izypower Titan en autoconsommation solaire, à partir de la puissance mesurée par mon compteur Shelly Pro EM.

### 1. Objectif

Optimiser l'autoconsommation solaire en régulant automatiquement la charge et la décharge de la batterie selon la puissance échangée avec le réseau, la production photovoltaïque, le niveau de charge (SOC), le mode sélectionné et l'autorisation de fonctionnement.

Le blueprint doit être propre, lisible, commenté en français et compatible avec les versions récentes de Home Assistant.

### 2. Entités à rendre configurables

Toutes les entités doivent être sélectionnables dans l'interface Home Assistant lors de la création d'une automatisation à partir du blueprint.

- Capteur de puissance réseau Shelly Pro EM : `sensor.shellyproem50_08f9e0e6a638_energy_meter_0_power`
- Capteur de SOC batterie : `sensor.izypower_titan_192_168_1_75_titan_battery`
- Capteur de production solaire : `sensor.192_168_1_65_power`
- Sélecteur de mode : `input_select.action_batterie`
- Autorisation de fonctionnement : `input_boolean.batterie_autorise`
- Consigne de puissance batterie : `number.izypower_titan_charge_discharge_power`
- Puissance maximale de charge : `number.izypower_titan_max_charge_power`
- Puissance maximale de décharge : `number.izypower_titan_max_discharge_power`
- Bouton de démarrage de charge : `button.izypower_titan_start_charge`
- Bouton de démarrage de décharge : `button.izypower_titan_start_discharge`
- Bouton de mise en veille : `button.izypower_titan_standby`

Utilise les sélecteurs de blueprint appropriés pour chaque type d'entité. Si nécessaire, ajoute des paramètres configurables pour les seuils, les puissances limites, les horaires et les temporisations.

### 3. Convention de puissance réseau

Respecte impérativement cette convention :

- Puissance réseau positive : la maison importe de l'électricité depuis le réseau.
- Puissance réseau négative : la maison injecte un surplus solaire sur le réseau.

Le système doit viser une légère importation réseau d'environ **+30 W**, avec une bande morte de **40 W**, afin de limiter les oscillations et les changements trop fréquents de consigne.

Vérifie que le sens de régulation est cohérent avec cette convention.

### 4. Modes de fonctionnement

Le sélecteur `input_select.action_batterie` possède trois états : `Charge`, `Decharge` et `Auto`.

**Mode Charge**
- Prioriser la récupération du surplus solaire.
- Augmenter la charge lorsque la puissance réseau devient négative.
- Réduire la charge si la maison importe trop de puissance.
- Éviter les variations brutales et les oscillations.
- Respecter la puissance maximale de charge configurée.

**Mode Decharge**
- Décharger la batterie de façon régulée pour réduire l'importation réseau.
- Ne pas dépasser la puissance maximale de décharge.
- Arrêter ou mettre en veille la batterie lorsque les conditions de fonctionnement ne sont plus satisfaites.

**Mode Auto**
- En journée, privilégier la charge à partir du surplus solaire disponible.
- La nuit, à partir de 23 h, autoriser une décharge limitée à 200 W maximum pour couvrir une partie des consommations domestiques importantes.
- N'autoriser cette décharge nocturne que si le SOC est supérieur à 70 %.
- Arrêter la décharge nocturne lorsque le SOC descend sous 20 %.
- En dehors de ces conditions, adopter un comportement sûr et cohérent, sans alterner rapidement entre charge et décharge.

Rends les seuils de SOC et les horaires configurables lorsque cela est pertinent.

### 5. Paramètres de régulation initiaux

Utilise les valeurs suivantes comme paramètres par défaut :

- Consigne réseau : +30 W.
- Bande morte : 40 W.
- Variation maximale de puissance par cycle : 50 W.
- Pas minimal de variation : 10 W.
- Puissance maximale de charge : 2 400 W, uniquement si cette limite est compatible avec les capacités réelles de la batterie et sa configuration.
- Puissance maximale de décharge : 500 W.
- Puissance maximale de décharge nocturne en mode Auto : 200 W.

Ne suppose pas que les limites théoriques sont nécessairement les limites autorisées par mon matériel. Signale clairement les paramètres qui doivent être vérifiés.

### 6. Particularité essentielle du pilotage Izypower Titan

La consigne numérique de puissance est toujours une valeur positive, que la batterie charge ou décharge.

Le sens du fonctionnement est sélectionné séparément à l'aide des boutons :
- `button.izypower_titan_start_charge`
- `button.izypower_titan_start_discharge`

**Règle impérative : à chaque modification de la consigne numérique de puissance, il faut appuyer sur le bouton correspondant au sens de fonctionnement souhaité.** Ne suppose pas que le sens reste automatiquement correct après une modification de consigne.

Organise les actions pour éviter les commandes contradictoires. Lors d'un changement de sens, applique une séquence sûre : réduis ou neutralise la consigne si nécessaire, attends si le protocole de l'appareil l'exige, sélectionne le nouveau sens, puis applique la nouvelle puissance. Explique les hypothèses retenues si le comportement exact du firmware n'est pas documenté.

N'envoie pas une commande de charge puis une commande de décharge dans le même cycle sans raison explicite.

### 7. Exigences techniques du blueprint

- Utilise la syntaxe officielle des blueprints Home Assistant.
- Déclare les entrées dans `blueprint:`, avec des sélecteurs adaptés.
- Utilise `!input` dans les champs compatibles, notamment pour les déclencheurs.
- Pour les modèles Jinja, expose les entrées nécessaires dans la section `variables:` puis utilise ces variables dans les templates.
- Ne crée pas de variables d'automatisation en espérant qu'elles soient disponibles dans les déclencheurs : les déclencheurs sont configurés avant l'exécution des actions.
- Vérifie les types des états et convertis les valeurs numériques avec prudence.
- Prévois des déclencheurs sur les changements de puissance réseau, de SOC, de production solaire, de mode et d'autorisation, ainsi qu'un déclencheur périodique pour corriger les dérives.
- Ajoute une temporisation ou une limitation de fréquence pour éviter les oscillations.
- Choisis un `mode` d'automatisation adapté afin d'éviter les exécutions concurrentes et les commandes contradictoires.
- Vérifie la disponibilité et la validité des capteurs avant d'envoyer des commandes.
- En cas de capteur indisponible, de valeur invalide ou d'autorisation désactivée, adopte un comportement sûr et documenté.
- Évite les boucles infinies et les commandes répétées inutiles.
- Ne suppose pas que le capteur de production solaire mesure la puissance totale des panneaux sans le signaler comme point à vérifier.
- Utilise le bouton de mise en veille uniquement lorsque son usage est justifié par la logique de fonctionnement.

### 8. Résultat attendu

Fournis dans cet ordre :

1. Le **fichier YAML complet du blueprint**, prêt à être copié dans un fichier sous `config/blueprints/automation/`.
2. Les instructions pour importer le blueprint et créer une automatisation depuis l'interface Home Assistant.
3. Une explication claire de la logique de régulation et du sens des puissances.
4. Un tableau récapitulatif des paramètres par défaut et de leur rôle.
5. Une liste des points à vérifier sur ma batterie avant la mise en service, notamment le comportement des boutons de charge/décharge et les limites de puissance.
6. Des exemples de fonctionnement pour les cas suivants : surplus solaire, importation réseau, changement de mode, SOC faible et décharge nocturne.
7. Une vérification de cohérence du YAML et des templates, en signalant toute hypothèse ou limitation qui empêcherait de garantir un fonctionnement correct sans test réel.

Ne fournis pas de pseudo-code ni de fragments incomplets. Si une fonctionnalité dépend d'un comportement spécifique du firmware Izypower Titan qui ne peut pas être confirmé, indique-le explicitement au lieu de l'inventer.
