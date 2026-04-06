## V1 – Mise en place d’une architecture Nios II / PIO

La première étape du projet a consisté à mettre en place une architecture minimale autour du processeur Nios II afin de valider la chaîne complète matériel / logiciel.

Cette première version repose sur un système simple composé d’un processeur Nios II et de deux interfaces PIO : une entrée reliée aux interrupteurs de la carte et une sortie reliée aux LED. L’objectif était de vérifier que le système Qsys était correctement généré, intégré dans le top-level VHDL, puis piloté par un programme embarqué.

Le programme C associé est volontairement élémentaire : il lit l’état des interrupteurs et le recopie en continu sur les LED. Cette boucle infinie permet de valider immédiatement le dialogue entre le processeur et les périphériques mémoire-mappés.

Cette étape a permis de confirmer le bon fonctionnement du système Nios II, le mappage correct des PIO et l’exécution d’un premier programme C sur le processeur. Elle constitue la base de tout le reste du projet.

## V2 – Intégration de la SDRAM

La deuxième version reprend l’architecture précédente en y ajoutant la mémoire SDRAM de la carte.

Le fonctionnement visible reste identique, avec une recopie de l’état des interrupteurs vers les LED, mais l’architecture matérielle est enrichie par l’ajout du contrôleur SDRAM et de tous les signaux associés (`DRAM_CLK`, `DRAM_CKE`, `DRAM_ADDR`, `DRAM_BA`, `DRAM_CS_N`, `DRAM_CAS_N`, `DRAM_RAS_N`, `DRAM_WE_N`, `DRAM_DQ`, `DRAM_DQM`).

Le programme C reste très simple, mais les adresses des PIO changent, ce qui montre que l’espace mémoire du système a évolué avec l’ajout de la SDRAM.

Cette version a permis de valider l’intégration correcte de la SDRAM dans l’architecture Nios II / PIO et de préparer le système à des versions plus complexes.

## V3 – Commande des moteurs et première caractérisation

La troisième version marque le passage vers une architecture réellement orientée robotique avec l’intégration de la commande des deux moteurs.

Le top-level `Robot.vhd` ajoute les signaux nécessaires au driver moteur de la carte robot, notamment `MTRR_N`, `MTRR_P`, `MTRL_N`, `MTRL_P`, ainsi que les signaux d’activation `MTR_Sleep_n` et `VCC3P3_PWRON_n`. Le système Qsys fournit alors deux sorties mémoire-mappées reliées au bloc `PWM_generation`, chargé de produire les signaux de commande des moteurs.

La commande moteur repose sur un mot de 14 bits : un bit `GO`, un bit `DIR` et 12 bits de vitesse. Cette structure a été exploitée dans plusieurs programmes C de test.

`Robot.c` a permis de valider le principe général de commande avec une même consigne appliquée aux deux moteurs. `Robot-air.c` a ensuite servi à faire varier progressivement la vitesse afin de déterminer le seuil minimal de mise en rotation hors sol. Enfin, `Robot-sol.c` a permis de vérifier le comportement du robot posé au sol avec une consigne fixe.

Cette étape a permis de valider la chaîne complète de commande moteur et de commencer la caractérisation expérimentale des moteurs, en identifiant les ordres de grandeur utiles pour la suite du projet.

## V4 – Calcul de la position de la ligne

La quatrième version introduit le calcul de la position de la ligne noire à partir des sept capteurs du robot.

Pour cela, l’architecture a été enrichie avec le bloc `position_ligne.vhd`, chargé d’analyser le vecteur seuillé des capteurs afin de déterminer deux informations : la présence de la ligne sous le robot et sa position relative par rapport au centre.

Le programme `position_ligne.c` lit les sept valeurs issues des capteurs via des adresses mémoire-mappées dédiées, puis les affiche sur le terminal JTAG UART. Cette visualisation a permis de vérifier expérimentalement le comportement des capteurs en fonction de la position du robot par rapport à la ligne.

Le calcul retenu fournit une position comprise entre `-6` et `+6`, ce qui permet d’obtenir une information plus fine qu’une simple indication gauche / droite. Cette donnée devient alors exploitable pour la correction de trajectoire dans les étapes suivantes.

Cette version constitue une transition importante entre la simple acquisition des capteurs et le pilotage intelligent du robot.

## V5 – Intégration du suivi de ligne et de la rotation

La cinquième version intègre les deux blocs principaux de commande du robot : l’automate de suivi de ligne `CTL_SL_final.vhd` et l’automate de rotation `CTL_ROT_final.vhd`.

Le top-level `position_ligne_nios_top.vhd` rassemble alors toute la chaîne de traitement : acquisition des capteurs, calcul de position, commande des moteurs, multiplexage des commandes de suivi et de rotation, ainsi qu’interface avec le processeur Nios II.

`CTL_SL_final.vhd` assure le suivi de ligne à partir de l’information de position. Il ajuste la vitesse des moteurs afin de corriger la trajectoire du robot et signale la perte de ligne par `fin_SL`.

`CTL_ROT_final.vhd` gère la rotation sur place. Lorsqu’il reçoit `start_rot`, il commande les moteurs jusqu’à retrouver la ligne et se recentrer, puis lève le signal `fin_rot`.

Le programme `nios_superviseur_portse_final.c` pilote ces blocs depuis le Nios II via deux ports mémoire-mappés : un port de commande (`start_sl`, `start_rot`, `dir_rot`) et un port d’état (`fin_sl`, `fin_rot`, présence de ligne, position courante). Cette version utilisait encore des temporisations franches afin d’obtenir un comportement stable et facilement observable lors des premiers essais.

Cette étape a permis de valider l’enchaînement entre perception, décision et action, avec un robot capable de passer automatiquement du suivi de ligne à la rotation.

## V6 – Gestion complète des aller-retours

La sixième version correspond à l’intégration complète du comportement d’aller-retour demandé dans le sujet.

L’architecture repose sur l’intégration des blocs `PosL_final.vhd`, `CTL_SL_final.vhd`, `CTL_ROT_final.vhd` et `position_ligne_nios_top.vhd`. À ce stade, le FPGA gère la logique temps réel liée aux capteurs et aux moteurs, tandis que le Nios II assure la supervision générale.

Le programme `nios_superviseur_portse_final.c` met en œuvre une machine d’états logicielle permettant d’enchaîner automatiquement les différentes phases : attente de la ligne, suivi, arrêt, rotation, puis reprise du suivi dans l’autre sens.

Le robot reste immobile tant qu’aucune ligne n’est détectée. Dès qu’elle apparaît, le suivi démarre. Lorsque la ligne est perdue, le robot s’arrête brièvement, lance la rotation, attend le recentrage, puis repart dans le sens opposé. L’inversion du sens de rotation après chaque demi-tour permet d’obtenir un véritable comportement d’aller-retour.

Cette version constitue la première version complète et autonome du système, capable d’enchaîner seul les différentes phases de déplacement.

## Version finale – Validation globale du système

La version finale correspond à l’intégration complète de l’architecture développée au fil du projet. Elle regroupe dans `position_ligne_nios_top.vhd` l’ensemble des blocs nécessaires au fonctionnement autonome du robot : calcul de position de la ligne, suivi de ligne, rotation sur place et supervision par le processeur Nios II.

Dans cette dernière version, le superviseur logiciel a été affiné afin d’obtenir un comportement plus souple et plus stable. Les grandes temporisations ont été remplacées par des phases d’acquittement et de courtes pauses de stabilisation avant et après la rotation, ce qui a permis d’améliorer la fluidité globale du système.

Les essais réalisés ont permis de valider un aller-retour complet sur une ligne droite : le robot suit la ligne, s’arrête en bout de ligne, effectue un demi-tour, se recentre puis repart dans le sens opposé. Le scénario principal attendu a donc été validé sur une trajectoire rectiligne.

En revanche, le suivi de ligne dans les virages n’a pas pu être validé correctement. Le système s’est montré fonctionnel sur une ligne droite, mais encore insuffisamment robuste lorsque la trajectoire comportait des changements de direction plus marqués.

La version finale permet néanmoins de conclure que les objectifs principaux du projet ont été atteints sur le cas d’usage essentiel. Le robot est capable de détecter une ligne, de la suivre, de s’arrêter à son extrémité, d’effectuer un demi-tour et de repartir automatiquement dans l’autre sens. L’architecture obtenue constitue ainsi une base solide pour de futures améliorations, en particulier sur le réglage du suivi de ligne dans les courbes et les virages.

## Annexe – Quelques notions de code

### `generic` en VHDL

En VHDL, les `generic` servent à paramétrer une entité sans modifier son architecture interne.  
Ils sont déclarés dans l’entité et fixés soit par une valeur par défaut, soit au moment de l’instanciation avec un `generic map`.

Dans notre projet, ils ont été utiles pour régler certains paramètres de fonctionnement comme les vitesses, les temporisations ou les seuils de validation. Par exemple, dans les blocs de suivi de ligne ou de rotation, un `generic` permet de modifier une vitesse de rotation ou un nombre de cycles sans avoir à réécrire le bloc.

L’intérêt est de rendre le code plus modulaire : un même composant VHDL peut être réutilisé avec plusieurs réglages selon le contexte.

### `inline` en C

En C, `inline` est utilisé pour les fonctions très courtes appelées fréquemment.  
Le compilateur peut alors remplacer l’appel de fonction par le contenu de la fonction directement, ce qui évite le surcoût d’un appel classique.

Dans notre projet, `inline` a surtout été utilisé pour les petites fonctions d’accès aux ports mémoire-mappés du Nios II, par exemple pour lire ou écrire rapidement un registre. Cela permet de garder un code lisible, avec des fonctions bien nommées, tout en conservant une exécution efficace dans les boucles de supervision.

L’écriture `static inline` signifie en plus que la fonction reste locale au fichier source, ce qui est adapté pour les fonctions utilitaires internes.