---

layout: article
title: Créer son propre bot discord
tags:
    - java
    - discord
    - bot
    - tutoriel

author: Rémi Lecouillard
---

Dans ce tutoriel nous allons voir les différentes étapes de création d'un *bot* discord. Je suppose que vous avez déjà un bagage en programmation, notamment en java. Nous nous attarderons pas sur le coté programmation pure du *bot*. Comme nous le verrons dans le tutoriel, il existe de nombreux langages et bibliothèques pour utiliser l'API de discord. Je laisse donc ce travail à la documentation de ces bibliothèques. Ici nous nous attarderons sur les grandes étapes pour avoir un *bot* fonctionnel.

## Création de votre compte

Le première étape est de... créer un compte discord évidemment. Pas besoin d'un compte développeur en particulier, un simple compte comme celui que vous utilisez est suffisant. Si vous n'en n'avez pas encore vous pouvez vous rendre sur https://discordapp.com/ pour en créer un. 

De l'extérieur un *bot* discord fonctionne comme un simple compte. Dans les faits, c'est n'est pas directement votre compte qui va être le *bot*. Depuis votre compte vous allez pouvoir créer et gérer différentes applications et ajouter un *bot* à ces applications. Libre à vous d’utiliser votre compte personnel pour gérer vos bots, ou de créer un compte spécifique.

### Créer votre *bot*
Pour créer une application il faut vous rendre sur cette page https://discordapp.com/developers/applications (vous devriez y être connecté). Comme vous pouvez le voir (image ci dessous), j'ai déjà une application nommé "Le Melting *bot*". Normalement vous ne devriez en avoir aucune, les différentes applications que vous aurez créée vont s'afficher ici. Cliquez sur *New Application* entouré en rouge sur l'image.

![interface création d'application](/assets/images/2020-03-19-bot-discord/approuge.png)

Donnez le nom que vous désirez à votre application. Allez ensuite dans l'onglet *bot* puis cliquez sur *Add bot* et... Voilà, il est créé. A partir de là, vous pouvez changer le nom de votre *bot* ou lui mettre une image de profil à votre guise. Dans ce tutoriel, en ces temps de confinement lié au coronavirus, notre *bot* d’exemple s’appellera "On peut sortir ?". Se sera un *bot* très simple qui répondra "non" dés que quelqu'un demandera si on peut sortir.

Vous devriez avoir en dessous du nom de votre *bot* écrit *token*. Affichez le, c'est cette identifiant qui vous permettra de prendre le contrôle de votre *bot* depuis l'API. Gardez le secret ! Je vous laisse explorer de vous même l'interface pour découvrir les différentes fonctionnalités.

### Ajouter votre *bot* à un serveur

Pour inviter votre *bot* à un serveur, vous aurez besoin de deux choses, de son identifiant (client id) et de ses autorisations sur le serveur. Pour le premier c'est très simple allez dans l'onglet *General Information* et copiez le numéro en dessous de client id. 
Pour les autorisations allez dans l'onglet *bot*, tout en bas dans *bot* Permissions. Cochez les permissions que vous aurez besoin pour votre *bot*. Dans notre cas nous avons uniquement besoin de *send messages*, mais on peut imaginer toutes sortes de *bot* qui peuvent gérer les membres, envoyer des fichiers, etc. Copiez ensuite le nombre qui est généré. 
Enfin, remplacez CLIENTID et PERMISSION dans ce lien par les deux numéros que vous venez de trouver. https://discordapp.com/oauth2/authorize?&client_id=CLIENTID&scope=bot&permissions=PERMISSION

Dans notre exemple le lien donne https://discordapp.com/oauth2/authorize?&client_id=690163488760529032&scope=bot&permissions=2048

Ouvrez tout simplement ce lien dans votre navigateur, vous pourrez alors choisir dans quel serveur l'ajouter. (Et oui vous pouvez ajouter notre exemple à votre serveur bande de petit coquin). Vous devriez normalement voir le message indiquant la venue de votre *bot*.

> Mais il est tout nul, il ne fait rien ce *bot* :'( 

Oui oui, on y vient, dans la partie suivante nous verrons comment piloter notre *bot* grâce à une bibliothèque qui utilise l'API de discord.

## Hello Discord

Dans ce tutoriel nous allons utiliser java, gradle et Intellij pour coder notre *bot*. A savoir qu'il est possible de coder un *bot* discord dans de nombreux langages (python, javascript, C++, etc.) . Libre à vous d’utiliser votre langage favoris. Vous pouvez trouver toutes les bibliothèques conformes à l'API discord ici : https://discordapp.com/developers/docs/topics/community-resources. 

### Création du projet et importation de la bibliothèque JDA

Créez un projet gradle avec Intellij, comme dit au début de ce tutoriel, je suppose que vous avez l'habitude de programmer donc je ne vais pas m'attarder la dessus.

    plugins {  
      id 'java'  
      id 'application'  
    }  
      
    group 'dev.jaaj.botdiscord'  
    version '1.0-SNAPSHOT'  
      
    sourceCompatibility = 1.11  
      
    repositories {  
      mavenCentral()  
      jcenter()  
    }  
      
    dependencies {  
      compile "net.dv8tion:JDA:4.1.1_101"  
    }	
    
    mainClassName = "Application"

Votre `build.gradle` devrait ressembler à quelque chose comme ça. Nous allons utiliser JDA (Java Discord API) pour piloter notre *bot*. Vous pouvez retrouvez le code source de cette bibliothèque ici https://github.com/DV8FromTheWorld/JDA. A noter que le dépôt *Maven* se trouve sur *jcenter*. Vous pouvez le trouver à cette adresse https://bintray.com/bintray/jcenter/net.dv8tion%3AJDA.

### Le code java

Passons maintenant au code minimal pour que  que notre *bot* réponde "Non." quand nous demandons si nous pouvons sortir. Votre fonction `main` devrait ressembler à quelque chose comme ça :
```java
public static void main(String[] argv) throws LoginException {
	JDABuilder builder = new JDABuilder(AccountType.*bot*);  
	builder.setToken(argv[1]);  
	builder.addEventListeners(new Application());  
	builder.build();  
}
```
Vous devez mettre en paramètre de `setToken` le *token* de votre application que nous avons vu plus haut. Il est préférable de le passer par les arguments du programme pour éviter d'écrire en dur le *token* dans le code (il doit resté secret). L'objet Application passé en argument de `addEventListeners` va être de type `ListenerAdapter`. Regardons à quoi il ressemble :

```java
public class Application extends ListenerAdapter {  
 
  @Override  
  public void onMessageReceived(@Nonnull MessageReceivedEvent event) {
	  if (event.getMessage().getContentRaw().toLowerCase().contains("on peut sortir ?")) {  
		  event.getChannel().sendMessage("Non. #RestezChezVous").queue();  
	}  
    }  
}
```
A partir de là, le code est assez explicite. Quand on reçoit un message, on regarde si le contenu correspond à "On peut sortir ?", on envoie ensuite dans le channel du message "Non. #RestezChezVous". Exécutez votre application, et testez la sur votre serveur. Dans notre exemple ça donne ça :

![messages du *bot*](/assets/images/2020-03-19-bot-discord/sortiiiir.PNG)


Voilà ! Votre *bot* discord est créé et fonctionnel. JDA propose beaucoup de fonctionnalités que je vous laisse découvrir dans sa documentation https://ci.dv8tion.net/job/JDA/javadoc/. Je vous recommande également d'aller lire le *readme* du code source. Il explique plus en détail comment utiliser la bibliothèque.

Comme dit au début du tutoriel, nous n'allons pas nous attarder sur comment coder le comportement du *bot* en java. Libre à vous d'aller lire la documentation des différentes bibliothèques et choisir laquelle utiliser.

## Déployer votre *bot*

> Mais je ne vais pas laisser mon PC allumé avec IntelliJ, comment je fais ?

Tout à fait. Dans notre exemple nous allons utiliser le plugin *shadow* de gradle pour exporter notre application java.  Il suffit de rajouter ces quelques lignes dans le `build.gradle`. Exécutez ensuite la commande `runShadow` pour générer votre jar, que vous pouvez récupérez dans `build/lib/`. 

	plugins {  
	  id 'com.github.johnrengelman.shadow' version '5.2.0'  
	}
	shadowJar {  
	  configurations = [project.configurations.compile]  
	}

Pour ma part, j'ai ensuite copié ce fichier sur un serveur linux et je l'ai lancé la commande : 

```bash
java -jar OnPeutSortir-1.0-SNAPSHOT-all.jar TOKEN &.
```

 Voilà ! Vous êtes libre de faire connaitre votre superbe *bot* au monde entier à présent.