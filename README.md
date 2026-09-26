# Simulation de moulage automobile V2 + dashboard Node-RED

Ligne de moulage simulée dans le navigateur, supervisée par un dashboard Node-RED (compteurs, TRS, température, taux de rebut, alertes et commandes à distance).

Fichiers à utiliser :

- `SimulationMoulageAuto_V2.html` : simulation
- `moulage-nodered-flow-v2.json` : flux à importer dans Node-RED

## Prérequis

- [Node.js](https://nodejs.org/) (LTS)
- Un navigateur récent
- Node-RED et le dashboard :

```powershell
npm install -g node-red
```

Puis, dans Node-RED : menu **Manage palette** → **Install** → `node-red-dashboard`.

## Lancer le TP

1. Démarrer Node-RED :

```powershell
node-red
```

2. Ouvrir l'éditeur : [http://127.0.0.1:1880](http://127.0.0.1:1880)
3. Menu **Import** → **select a file to import** → choisir `moulage-nodered-flow-v2.json` → **Import** → **Deploy**.
4. Ouvrir `SimulationMoulageAuto_V2.html` dans le navigateur (double-clic sur le fichier).
5. Descendre jusqu'au panneau **WebSocket Node-RED**, en bas de page, sous le tableau **Traçabilité**.
6. Laisser l'adresse `ws://127.0.0.1:1880/ws/moulage` et cliquer sur **Connecter**. Le texte doit passer à **Connecté**.
7. Ouvrir le dashboard : [http://127.0.0.1:1880/ui](http://127.0.0.1:1880/ui), onglet **Moulage V2**.
8. Cliquer sur **Démarrer** (sur la page ou sur le dashboard).

## Ce que montre le dashboard

| Zone | Contenu |
| --- | --- |
| État | Statut de la ligne, goulot d'emballage, mode Défi, alerte qualité |
| Compteurs | Pièces totales, conformes, rebuts, pièces emballées |
| Performance | Jauge TRS / OEE, jauge température moule, courbe du taux de rebut |
| Commandes | Démarrer, Arrêter, Réinitialiser Sécurité, vitesse, température, téléchargement CSV |

Le journal CSV est écrit ici :

`C:/Master/M2/IOT/TP/Production/moulage_production_v2.csv`

Colonnes : `ID_Piece`, `Horodatage`, `Temp_Moule`, `Statut`, `Motif_Rebut`, `Emballe`.

Si le projet n'est pas dans ce dossier sur ta machine, ouvre les nœuds **Journal production V2** et **Lire le journal CSV** dans Node-RED et change le chemin du fichier avant de déployer.

Sur le dashboard, le bouton **Télécharger le CSV** du groupe Commandes récupère ce fichier. La même adresse fonctionne dans le navigateur : [http://127.0.0.1:1880/moulage/production.csv](http://127.0.0.1:1880/moulage/production.csv). Réimporte `moulage-nodered-flow-v2.json` puis **Deploy** si ce bouton n'apparaît pas encore.

## Commandes envoyées par le dashboard

La page reçoit du JSON sur le WebSocket :

- Démarrer → `{ "action": "START" }`
- Arrêter → `{ "action": "STOP" }`
- Réinitialiser Sécurité → `{ "action": "RESET" }`
- Vitesse du convoyeur (10 à 100) → `{ "action": "SET_SPEED", "value": 60 }`
- Température (150 à 250 °C) → `{ "action": "SET_TEMP", "value": 185 }`

La simulation traite `START`, `STOP`, `RESET` et `SET_SPEED`. Le curseur de température du dashboard envoie `SET_TEMP`, mais la page ne l'applique pas encore : la température se règle avec le curseur **Température de la presse** sur la simulation.

## Comportements à observer

- **Mode Défi** (bouton sur la page) : produire 100 pièces conformes emballées en moins de 10 minutes. Le chrono en haut du synoptique compte le temps de session ; pendant le défi, il devient un compte à rebours de 10:00.
- **Surchauffe aléatoire** : après **Démarrer**, un instant est tiré au hasard dans les 10 minutes de marche. La température passe alors entre 225 et 245 °C, le voyant **Surchauffe** s'allume et la ligne s'arrête. **Arrêter** met ce délai en pause. Pour repartir : baisser la température sous 220 °C, puis **Réinitialiser**.
- **Alerte qualité** : si le taux de rebut dépasse 30 %, le dashboard affiche **Alerte Qualité Majeure**.
- **Arrêt qualité** : 3 rebuts consécutifs arrêtent la ligne. Acquitter avec **Réinitialiser Sécurité** sur le dashboard, ou **Réinitialiser** sur la page.
