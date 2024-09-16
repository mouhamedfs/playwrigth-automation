# 📚 Ce dépôt est utilisé pour apprendre et développer de nouvelles compétences avec Playwright

Ce repôt contient des fichiers de configuration pour exécuter des tests automatisés Playwright à l'aide de **GitHub Actions**. Il permet de :
- 🔍 Tester les navigateurs avec Playwright
- 🚀 Générer des rapports de tests avec **Allure**
- 🔔 Envoyer des notifications Slack après chaque exécution de test

---
## 🌐 Configuration de GitHub Pages pour Allure

Pour visualiser les rapports **Allure** via **GitHub Pages**, suivez ces étapes pour configurer votre dépôt et publier le rapport.

### 1. Activer GitHub Pages

1. **Accéder aux paramètres de votre dépôt** :
    - Allez dans la section **Settings** de votre dépôt GitHub.

2. **Configurer GitHub Pages** :
    - Dans la section **Pages** (visible sous l'onglet **Code et automatisation** ou **Pages**), sélectionnez la branche `gh-pages` dans le menu déroulant **Branch**. Si cette branche n'existe pas encore, elle sera automatiquement créée lorsque vous lancerez le workflow.
    - Sélectionnez le dossier racine `/ (root)` et cliquez sur **Save**.

3. **Obtenir le lien GitHub Pages** :
    - Après avoir configuré GitHub Pages, un lien vers votre site sera généré. Par exemple : `https://[nom-utilisateur].github.io/[nom-du-repo]/`.
    - Ce lien sera utilisé pour accéder aux rapports Allure générés après chaque exécution des tests.

### 2. Publier le Rapport Allure via GitHub Actions

Votre workflow GitHub Actions contient déjà l'étape suivante pour publier le rapport Allure sur GitHub Pages :

```yaml
- name: Upload Allure Report to GitHub Pages
  uses: peaceiris/actions-gh-pages@v3
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: ./allure-report
```

## 🔔 Configuration des Notifications Slack

Pour recevoir des notifications via Slack chaque fois qu'un test est exécuté (avec un lien vers le rapport Allure), vous devez configurer un webhook Slack et un workspace Slack.


1. **Créer un Webhook Slack** :
    - Créer un Workspace Slack :
    - Si vous n'avez pas encore de workspace Slack, vous pouvez en créer un en allant sur **Slack**.
    - Suivez les étapes pour créer un compte et configurer votre propre espace de travail.



2. **Configurer un Webhook Slack**:
    - Accédez à la section Apps de votre workspace Slack et recherchez l'application **Incoming Webhooks**.
    - Cliquez sur Ajouter à Slack.
    - Sélectionnez le canal où vous souhaitez recevoir les notifications, puis cliquez sur Autoriser.
    - Une fois ajouté, **un URL de webhook** vous sera fourni. Copiez cet URL.


3. **Ajouter le Webhook dans GitHub**:
    - Allez dans les Settings de votre dépôt GitHub.
    - Sous Secrets and variables > Actions, cliquez sur New repository secret.
    - Ajoutez une nouvelle clé nommée **SLACK_WEBHOOK_URL** et collez l'URL du webhook que vous avez copié.



## ⚙️ GitHub Actions : Exécution des tests Playwright

Le fichier YAML suivant permet d'exécuter les tests Playwright automatiquement lors d'une **pull request** ou d'un **push** sur les branches `main` et `master`.

### 🎯 Déclencheurs
- **Push** sur les branches `main` et `master`
- **Pull Request** sur les branches `main` et `master`

---


## 📋 Documentation des Jobs

### **1. Job : Test**

Ce job est responsable de l'exécution des tests Playwright dans un environnement Linux avec une configuration Ubuntu à jour. Voici les étapes principales :

1. **Cloner le dépôt** :
    - Utilise l'action `actions/checkout@v2` pour cloner le dépôt GitHub dans l'environnement de travail.

2. **Installer Node.js** :
    - Utilise `actions/setup-node@v4` pour configurer l'environnement Node.js avec la version LTS (support à long terme). Cela garantit que Playwright et ses dépendances sont exécutés dans une version stable de Node.js.

3. **Installer les dépendances** :
    - Utilise `npm ci` pour installer les dépendances du projet (cela garantit des installations rapides et reproductibles).

4. **Installer les navigateurs Playwright** :
    - Utilise la commande `npx playwright install --with-deps` pour installer les navigateurs nécessaires pour les tests.

5. **Exécuter les tests Playwright** :
    - Lance les tests avec `npx playwright test`, et passe des variables d'environnement (`XRAY_URL`, `XRAY_CLIENT_ID`, `XRAY_CLIENT_SECRET`) à partir des secrets GitHub pour une éventuelle intégration avec Xray ou d'autres services de gestion des tests.

6. **Générer un rapport Allure** :
    - Après les tests, génère un rapport Allure détaillé avec `npx allure generate allure-results --clean -o allure-report`. Ce rapport contient les résultats des tests et leur couverture.

7. **Publier le rapport Allure sur GitHub Pages** :
    - Utilise l'action `peaceiris/actions-gh-pages@v3` pour publier le rapport Allure dans GitHub Pages, accessible via un lien public.

8. **Télécharger le rapport Allure en tant qu'artifact** :
    - Sauvegarde le rapport Allure avec `actions/upload-artifact@v2` pour que les utilisateurs puissent le consulter directement depuis l'interface GitHub.

### **2. Job : Notification**

Ce job se déclenche toujours après l'exécution du job `test`, qu'il soit réussi ou échoué, et envoie une notification à Slack pour indiquer l'état des tests. Voici comment cela fonctionne :

1. **Condition d'exécution** :
    - Ce job dépend de la fin du job `test` grâce à `needs: test`, et il est configuré pour s'exécuter même en cas d'échec de `test` grâce à `if: always()`.

2. **Notification via Slack** :
    - Utilise une commande `curl` pour envoyer une requête POST au webhook Slack. Le message contient un texte personnalisé indiquant si les tests ont réussi ou échoué :
        - En cas de succès, le message est vert avec `"Les tests Playwright ont réussi avec succès !"`.
        - En cas d'échec, le message est rouge avec `"Les tests Playwright ont échoué. Veuillez vérifier les résultats."`.

3. **Lien vers le Rapport Allure** :
    - La notification inclut également un lien vers le rapport Allure hébergé sur GitHub Pages, afin que les utilisateurs puissent consulter les détails des tests.

```yaml
      - name: Send Slack Notification
        env:
         SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        run: |
         STATUS=${{ needs.test.result }}  # Test the result status (success or failure)
         if [[ "$STATUS" == "success" ]]; then
           COLOR="#36a64f"
           MESSAGE="Playwright tests have passed successfully!"
         else
           COLOR="#ff0000"
           MESSAGE="Playwright tests failed. Please review the results."
         fi
         # Send the notification to Slack
         curl -X POST -H 'Content-type: application/json' \
         --data '{
           "attachments": [
             {
               "color": "'"$COLOR"'",
               "pretext": "Test Notification from GitHub Actions",
               "text": "'"$MESSAGE"'",
               "fields": [
                 {
                   "title": "Allure Report",
                   "value": "<https://mouhamedfs.github.io/playwrightExpert/|Allure Test Report>",
                   "short": false
                 }
               ]
             }
           ]
         }' $SLACK_WEBHOOK_URL
```

## 🔔 Résultat des Notifications


Si les tests réussissent, vous recevrez un message vert dans Slack avec le texte :
"Les tests Playwright ont réussi avec succès !"

Si les tests échouent, vous recevrez un message rouge :
"Les tests Playwright ont échoué. Veuillez vérifier les résultats."

Chaque message contiendra un lien vers le rapport Allure publié sur GitHub Pages, permettant de visualiser les résultats détaillés des tests.