# Brancher Dilaya à ton IA

Tu pilotes Dilaya **depuis ton IA** : tu discutes avec elle, et Dilaya lui donne les mains pour construire ton logiciel, y ranger tes informations et le mettre en ligne. Ton logiciel, lui, tourne dehors : il a sa propre adresse et continue de tourner quand tu fermes la conversation. Pour ça, il faut relier les deux **une seule fois**. Ça prend deux minutes, et il n'y a rien à installer sur ton ordinateur.

**Marche avec Claude et ChatGPT.** Il vous faut un abonnement à votre IA, payé à part : Claude Pro ou ChatGPT Plus minimum.

## Dans Claude

Dilaya s'installe comme un **plugin** : rien à copier, rien à configurer.

1. Dans Claude (claude.ai ou l'application), ouvre **« Personnaliser »** puis **« Plugins »**, cherche **Dilaya** et clique sur **« Ajouter »**. Tu peux aussi y aller depuis le bouton « Brancher Dilaya » de ton espace client sur dilaya.eu.
2. Dans l'onglet **« Connecteurs »** du plugin, clique sur **« Connecter »** à côté de Dilaya : la page d'autorisation dilaya.eu s'ouvre (voir plus bas).
3. Dans les **réglages** de Claude, tout en bas, **« Domaines autorisés »** : choisis **« Gestionnaires de paquets »** et ajoute `*.dilaya.eu`. Dans une organisation **Claude Team ou Enterprise**, c'est le responsable de l'organisation qui fait cette étape.

Sur **téléphone**, le plugin se retrouve au même endroit ; si tu ne le vois pas, fais l'installation une fois depuis un ordinateur : elle vaut pour tous tes appareils.

## Dans ChatGPT

Dilaya s'ajoute comme un **serveur MCP personnalisé** (ChatGPT Plus minimum recommandé). Fais-le depuis le **navigateur** ou l'**application de bureau** : l'application mobile ne permet pas d'ajouter un serveur MCP personnalisé.

1. Ouvre **chatgpt.com/plugins?view=personal&directoryTab=personal** (la page Plugins, onglet **« Personnel »**), clique sur **« + »** puis **« Ajouter un serveur MCP personnalisé »** (« Add custom MCP server » si ton ChatGPT est en anglais).
2. Nom : **Dilaya**.
3. Connexion : **« Adresse publique »** (« Public endpoint ») → colle `https://app.dilaya.eu/mcp`.
4. L'authentification est détectée toute seule : la page d'autorisation dilaya.eu s'ouvre (voir plus bas).
5. Accepte l'avertissement (**« Je comprends et je souhaite continuer »**, « I understand and want to continue ») puis **« Créer en tant que plugin »** (« Create as a plugin »).
6. Ouvre une **nouvelle conversation** et dis : « Utilise l'outil guide de Dilaya ».

## Autoriser l'accès (la page dilaya.eu)

- **Connecte-toi** avec l'adresse e-mail de ton compte Dilaya : tu reçois un **code par e-mail**, pas de mot de passe à retenir. Pas encore de compte ? C'est ce code qui le crée.
- **Coche ton organisation** (ton atelier) : celle dont ton IA a le droit d'utiliser les logiciels. Si tu en as plusieurs, coche toutes celles que tu veux. Pas encore d'organisation ? La page te propose de **la créer sur place** : un nom, un identifiant, et c'est fait.
- Clique sur **« Approuver »**.

Tu reviens dans la conversation : c'est branché, et tu n'auras plus à le refaire.

## Dis ce que tu veux

Ouvre une conversation et parle normalement. Par exemple :

> « Avec Dilaya, je veux un carnet pour suivre mes clients et leurs commandes. »

Ton IA reconnaît Dilaya et se met au travail.

## Questions fréquentes

**Quelle offre me faut-il ?** Claude Pro ou ChatGPT Plus minimum. C'est une recommandation de qualité, pas une limite technique : en dessous, le branchement marche, mais l'IA n'a pas accès au mode qui lui permet de mener un vrai travail du début à la fin.

**Et si je change d'IA ?** Vos logiciels, vos données, vos fichiers et leurs consignes restent chez Dilaya. Vous branchez votre nouvelle IA, et elle reprend là où l'autre s'était arrêtée. Suivent : tes logiciels, tes données, tes fichiers, tes sites, tes consignes, ton bot Telegram et tes automatisations à heure fixe. Restent liés à Claude : les tableaux de bord vivants (Claude Cowork) et l'agent code (Claude Code).

**Une autre IA ?** Toute IA capable d'ajouter un serveur MCP peut se brancher à Dilaya avec la même page d'autorisation. Demande-nous la marche à suivre.

**Plusieurs personnes dans le même espace ?** Chacune branche Dilaya dans **son** IA, avec **son** compte Dilaya, une fois invitée dans l'espace.

**Débrancher ?** Dans Claude : « Personnaliser » → « Plugins », puis retire Dilaya. Dans ChatGPT : la page Plugins, puis retire le serveur Dilaya. Ton logiciel et tes informations restent en sécurité chez Dilaya ; tu peux rebrancher quand tu veux.
