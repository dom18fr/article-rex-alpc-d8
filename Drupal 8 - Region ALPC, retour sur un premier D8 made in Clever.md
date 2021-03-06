# Drupal 8 - Region ALPC, retour sur un premier D8 made in Clever

Petit tour d'horizon des nouveautés D8 (et de leurs petits pièges) qui se sont trouvées sur notre route lors de la réalisation de [http://www.laregion-alpc.fr](http://www.laregion-alpc.fr)

## 1 - Symfony2 & POO

Drupal 8 utilise beaucoup de composants Symfony. Pour autant ce n'est pas une application Symfony full stack (comme EZ publish 5/6 par exemple). Un certain nombre de concepts (notament dans la modélisation des contenus) n'ont pas été "symfonyfiés" (ce mot existe. Si, c'est vrai.).  

Cependant, la totalité de la base de code du cœur (et de la grande majorité des modules contrib portés et stables en D8) a été repensée / réécrite en POO.  

Pour développer en Drupal 8, sans être obligé de savoir vraiment faire du Symfony, il est tout de même préférable de maîtriser les notions de :

  * Injection de dépendances et services symfony
  * Structure et syntaxe des fichiers de conf en .yml
  * Annotations
  * Syntaxe TWIG

Plus généralement, il faut avoir les bases de la POO php :

  * Classes abstraites et héritage
  * Interfaces
  * Méthodes static / protected / public

Même si ça à l'air d'un gros morceau à avaler pour qui n'aurait fait que du drupal 6/7 jusque là (et donc quasiment pas de POO), ça n'a rien d'insurmontable. Ça n'a en tout cas pas posé de problème particulier sur ALPC, parmi les devs il y avait 1 Symfoniste confirmé et 2 drupalistes.

## 2 - Les blocs

### 2.1 - Plugin baby !

En Drupal 8, la notion de bloc a pas mal évolué en Drupal 8, tant dans les concepts que dans le code.

En D7 un bloc est un fragment de contenu, déclaré dans un hook et disponible pour être placé dans une région du thème. Un bloc donné ne peut apparaître qu'en un seul exemplaire et dans une seule région.
Ce mécanisme limité peut être étendu en utilisant des modules contrib comme context, panels ou display suite.

Drupal 8 a intégré la souplesse de gestion des blocs apportée par les modules cités plus haut dans le cœur en utilisant le duo plugin + config entity.

Le plugin est d'abord défini dans le code, puis une fois le plugin *block* disponible dans l'interface, on peut le "placer" dans une région pour un thème donné, surcharger son titre et ajuster les settings (visibilité par exemple). C'est à ce moment là qu'une instance de config entity est créée en base. La config entity contient toutes les infos relatives au placement du plugin *block*.

On peut créer plusieurs config entities pour un même plugin, avec des settings différents à chaque fois.

### 2.2 - Custom block (block_content)

En Drupal 7 il n'est pas possible nativement de créer des blocs configurables auxquels on attache des champs. C'est un besoin assez courant auquel répondent les modules contrib `nodeblock` et plus récemment (et plus proprement) `bean`.  

Là encore drupal 8 a intégré cette problématique dans le cœur. Il s'agit quasiment d'un port du module `bean` sous le nom de *Custom Block* (nom machine `block_content`).

Le module *Custom Block* definit le type d'entité `block_content`. Il est possible de créer des types de blocs (analogues aux type de contenus) auxquels on attache des champs pour la contribution. Ensuite, chaque `block_content` créé donne lieu à la création d'un plugin *block*. Il est donc possible de placer un bloc contribué comme un bloc définit dans le code.

Par exemple sur ALPC, les blocs image cliquables visibles dans le menu principal, dans la sidebar des articles et au bas de la page d'accueil sont des `block_content`.

## 3 - Form modes

Le form mode est l'équivalent du view_mode côté formulaire. Autrement dit il s'agit d'une structure qui stocke, pour un bundle donné (type de contenu, type de bloc etc...), la visibilité, l'ordre d'apparition et le widget de chaque champs.  

Il devient donc possible de créer plusieurs visualisations d'un même formulaire d'entité.  

Su ALPC nous l'avons peu utilisé, le seul exemple est le formulaire d'inscription à la newsletter qui apparaît sous une forme différente en homepage et sur les articles.

## 4 - {{ TWIG }}

### 4.1 - Render arrays & logique d'assemblage

D8 intégre Twig en lieu et place de phpTemplate. Et c'est bien.  
Le Twig embarqué dans Drupal est un Twig étendu, enrichi de nombreuses fonctions dédiées.
Contrairement à phpTemplate, Twig est intimement lié aux mécanismes de rendu de Drupal. Concrètement cela signifie que Twig peut afficher des chaînes, comme il le fait naturellement au sein de n'importe quel framework, et est également capable d'invoquer, de manière transparente la *render API* de Drupal. Autrement dit, Twig peut rendre/afficher des render arrays.

Par exemple dans un template de node comme :

```twig
<article>
  <h1>{{ title }}</h1>
  {{ content }}
</article>
```

`title` contient une chaîne et `content` contient un render array (incluant tous les champs) produit par le view_mode du node.

Le fait que Twig sache gérer les render array implique qu'une grande partie de la logique d'assemblage des templates se fait dans les render arrays. Ce n'est pas (ou rarement) au sein d'un template que l'on va appeler un sous template avec un include.  

Dans des cas d'assemblage 1 template parent / multiple templates enfants, comme typiquement le template de node vu plus haut, le template parent "reçoit" un render array contenant des enfants (les champs du node par exemple), et l'affiche tout simplement. La render API reprend alors la main et fait appel aux templates des champs pour résoudre le render array.

En bref, Twig ne porte pas la logique d'assemblage contrairement à l'usage qui peut en être fait dans d'autres frameworks.

Les avantages sont multiples :

  * Simplicité des templates : ils sont nombreux (si peu mutualisés) mais très atomiques, donc courts et lisibles.
  * Maintient complet de la logique de suggestions de templates, chaque template de chaque élement aussi petit soit-il a son propre template, lequel peut être choisi en fonction du contexte parmis un ensemble par convention de nommage. Ce qui rend le système très flexible.
  * Si la logique d'assemblage n'est pas écrite en dur dans les templates, il devient possible d'exposer à la contribution des mécanismes de layouts avancés (view_modes, panels, paragraphs etc ...)

Exemple d'un ensemble de templates pour le rendu d'un node dans le projet ALPC :

Arborescence :

```
node
|__ entity_wrapper
    |__ node--alpc-article.html.twig
|__ field
    |__ field--field-alpc-article-tags.html.twig
    |__ field--field-alpc-article-location.html.twig

```

*node--alpc-article.html.twig*

```twig
{{ attach_library('alpc/alpc-article') }}
{{ content }}
```

*field--field-alpc-article-tags.html.twig*

```twig
<ul id="tags" class="tags">
  {% for tag in items %}
    <li>
      {{ tag.content }}
    </li>
  {% endfor %}
</ul>
```

*field--field-alpc-article-location.html.twig*

```twig
<h3>{{ items|first.content }}</h3>
```

Les templates peuvent donc être peu nombreux et tres génériques, un seul template peut servir à rendre tous les champs d'une entité, ou tous les éléments d'un formulaire. Mais ils peuvent aussi nombreux et tres spécifiques chaque champ peut avoir sa logique d'intégration propre.

Pour celà Drupal 8 utilise un systeme de convention de nommage des templates.
par exemple : `field.html.twig` est le nom du template générique pour tous les champs, de toutes les entités, `field--field-alpc-article-location.html.twig` est le template spécifique du champ dont le nom machine est `field_alpc_article_location`. Le template chargé par Drupal sera toujours le plus spécifique disponible.

La convention de nommage qui permet d'utiliser le nom d'un champ est native, mais il arrive frequement que l'on souhaite dériver un template sur la base d'autres éléments du contexte. Pour résoudre ce problème, sur ALPC nous avons utilisé le theme de base [Cleverdrop](https://github.com/dom18fr/cleverdrop). Ses 2 rôles principaux sont :

  * Enrichir les convention de nommages de templates, permettant ainsi une plus grane spécificité des templates.
  * Fournir des templates générique simplifiés (plus légers que ceux fournis par Drupal nativement)

Il devient ainsi facile de reproduire n'importe quel intégration statique, sans écrire de php, simplement en créant et nommant des templates d'elements.

### 4.2 - Attributes

Le Twig de Drupal inclut également la gestion d'une objet de type `Attributes`. Il s'agit d'une structure portant des attributs html. Cet objet expose de multiples méthodes permettant de manipuler des attributs à la volée au moment de les afficher.

Cet objet Attributes permet de transmettre au templates des attributs définits en amont (au niveau d'un module par exemple) tout en laissant la ligerté au front d'y ajouter ceux qui relevent de la logique de theming, typiquement des classes css.

En effet dans la logique de séparation theme / module de drupal, tout le html n'est pas laissé à la responsabilité du thème. Le thème n'a qu'un rôle cosmetique. Certains attributs ou balises html peuvent être nécessaire fonctionnelement, comme par exemple les attributs des éléments de formulaire (action, method, name etc ...). Ceux là doivent pouvoir être définits dans un module et passé aux templates du thème.

Par exemple, le template des titres secondaires dans les articles d'ALPC :

```twig
<section class="second-title">
  {% set classes = ['title-outerline-blue', 'font-title'] %}
  <h2{{ attributes.addClass(classes) }}>{{ value }}</h2>
</section>
```

Les titres secondaires des pages doivent avoir un attribut id, de manière à être navigables par ancres. L'attribut id de chaque titre est donc créé par un module  et se retrouve dans l'objet attributes.


### 4.4 - Laissons les dev front travailler et les contributeurs contribuer !

D'une manière générale l'intégration de Twig et son imbrication dans la render API de Drupal permet une grande souplesse de contribution, y compris s'agissant du layout des éléments, et ce sans franchir la frontière (interdite) de la contribution de html et css en back-office. Il devient possible (plus encore qu'en D7) de déléguer aux contributeurs la construction du puzzle et aux dev front la responsabilité du design des pièces.

Le thème Drupal peut désormais être entièrement géré par les dev front, sans connaissances particulières de Drupal ou même de PHP.  
Là où en D7 une tres large partie du code PHP custom d'un projet se trouvait dans le thème, en D8 le thème peut (doit) être constitué exclusivement de Templates Twig, de feuilles de style, de fichiers js et d'assets (images, svg, json etc...)

Amis intégrateurs, les seules prérequis sont :

  * Accepter que la logique d'assemblage n'est pas (ou peu) dans les templates.
  * Connaître les grands principes de cette logique d'assemblage (tres atomisé).
  * Connaître les quelques extensions Twig spécifiques à Drupal.

Sur ALPC, pour des raisons de planning principalement, nous n'avons pas pu mettre en œuvre ce principe, ce sont les dev Drupal qui se sont chargés d'écrire les templates à partir d'une intégration statique réalisée en amont. Par ailleurs, le thème ALPC embarque malheureusement quelques lignes de code PHP.  
Mais la prochaine fois c'est la bonne :)


## 5 - Composer

### 5.1 - Composer dans le core de D8 !

Parmi les nouveautés D8, on trouve l'intégration de Composer. On est encore au début de l'histoire mais c'est déjà prometteur.
En réalité c'est surtout le cœur de Drupal 8 qui utilise Composer pour ses dépendances, mais lorsque l'on télécharge le cœur de Drupal, on récupère un ensemble de sources, incluant les dépendances du cœur dans un répertoire vendor.

Toutefois, il est possible d'ajouter des dépendances dans le fichier `composer.json` prévu à cet effet. Sur ALPC, c'est ce que nous avons fait, pour les libs ElasticSearch et pour les libs des API des réseaux sociaux.

### 5.2 - Bien mais pas top

Mais cela a posé un problème dans les procedures de mises à jour. En effet lors d'une mise à jour du cœur (sans Composer) on récupere toutes les sources, y compris le répertoire vendor qui ne contient pas nos dependances ajoutées, il faut donc faire un backup et merger tout ça à la main, pas très pratique.

Pour ALPC, vu le faible nombre de dépendances spécifiques du projet, on a fait avec. Nous avons utilisé Composer sur les postes de dev uniquement et versionné tous les vendor.  
Lors des mises à jour du cœur, on a fait attention (pour la vérité historique, j'ai d'abord tout cassé sans rien comprendre, puis j'ai réfléchi, et ensuite j'ai fais attention).

### 5.3 - drupal-project

Pour les prochaines fois, une solution complète est proposée par ce projet :
[https://github.com/drupal-composer/drupal-project](https://github.com/drupal-composer/drupal-project)  
Dans cette approche full composer, toutes les dépendances du projet (incluant le cœur de Drupal et les modules contrib) sont gérées par Composer, ce qui permet de ne versionner que le nécessaire et d'automatiser les mises à jour et les déploiements.

Il y aura peut être un article KB sur cet outil prochainement.

## 6 - Contact form

### 6.1 - Des formulaires d'entité

D8 introduit un nouveau modèle de formulaires de contact. Les contact forms de D8 sont contribuables, il est possible de définir les champs du formulaire depuis le BO.

Dans l'histoire de Drupal (depuis D7 en tout cas), 2 approches différentes ont existé pour créer des formulaires en BO.

* Le modèle "Webform", qui fournit des types de champs pour sa propre API, gère la construction du formulaire, son rendu en front, sa logique de validation et de soumission ainsi que la persistance des données soumises de manière spécifique.
* Le modèle "EntityForm", qui s'appuie sur les concepts d'entité et de champs existants dans le cœur de Drupal. En effet, les formulaires utilisés pour la contribution des entités sont finalement des formulaires dont on gère la structure en back-office. C'est encore plus vrai avec l'arrivé des form modes de D8.

Contact Form est un module du cœur de Drupal 8 qui s'inscrit dans la continuité de EntityForm, les formulaires de contact sont en fait des entités dont on expose le formulaire d'édition en front.

Cette approche permet une utilisation totale des concepts liés aux champs des entités (notions de widget, validateurs, rendu, form modes, theming, stockage en base etc ...)

A noter toutefois que la persistance des données en base n'est pas (encore) native. Par défaut les formulaires ne font qu'envoyer un mail comme le fait le module Contact dans Drupal 7. Le module contrib `contact_storage` complète `contact_form` pour disposer de véritables entités persistantes.

Sur ALPC nous avons utilisé des contact forms pour tous les formulaires exposés en front. Le formulaire de contact bien sûr, mais également le formulaire d'inscription à la newsletter, le formulaire d'inscription aux notifications d'agenda et communiqués de presse, un formulaire de recherche rapide et le formulaire de dépôt d'offres d'emploi.

Ces formulaires sont très différents et pour certains assez ambitieux niveau front / UX :

  * Soumission / Validation AJAX
  * Visibilité des champs modifiés à la volée
  * Autocomplete branché sur ElasticSearch
  * Affichage en popin
  * Intégration HTML / CSS 100% custom

Dans l'ensemble, contact form nous a permis de faire tout ça, en écrivant un peu de code, mais toujours en exploitant les API natives de Drupal, pas de JS ou de PHP custom qui réinvente la roue.

### 6.2 - Too much

Contact form en fait même un peu trop au bout du compte. En effet le module a vraiment été conçu pour des formulaires de contact. Il existe donc par défaut des champs qui ne sont pas utiles dans le cadre des formulaires d'un autre type. Ce n'est pas très grave car on peut utiliser les form modes pour choisir les champs que l'on souhaite exploiter.  

Plus gênant en revanche, la soumission d'un formulaire de contact **doit** envoyer un mail, c'est en dur dans le cœur, et de surcroît envoyer un mail à une adresse dont la saisie est obligatoire lors de la définition du formulaire.  
Dans le cas d'ALPC le seul formulaires dont on avait besoin qu'il envoi effectivement des mails (le formulaire de contact du site) devait exposer un champ dans lequel le visiteur pourrait choisir un service de la région comme destinataire de son message. Impossible donc d'exploiter le mécanisme natif d'envoi de mail.  
Il a donc fallu empêcher le mail de partir, et réimplémenter une logique d'envoi de mails parallèlement.  
(A cette occasion, nous avons essayé de nous appuyer sur `rules` pour le déclenchement de l'envoi du mail, sans succès. `Rules` est encore bien trop instable.)

### 6.3 - Webform ? Yamlform ? Eform ?

Avec le recul, ce genre de petits soucis me font penser que `contact_form` devra évoluer vers plus de souplesse à l'avenir et intégrer nativement `contact_storage` pour que son usage devienne aussi courant que l'était webform en D6 et D7.

Une autre alternative, si `contact_form` n'étend pas son périmètre, serait d'utiliser le module `eform` (le nom du projet `entityform` porté en D8), dont la vocation est de couvrir complètement le besoin des formulaires contribués.

A noter enfin que webform ne sera probablement pas porté en D8, mais qu'il existe le projet `yamlform`, semblable dans l'approche (n'utilisant pas les entités et leur champs, mais sa propre API).

## 7 - Migrate

Le module `Migrate` est dans le cœur de Drupal !  
Et c'est une bonne nouvelle. Cependant, à l'heure actuelle (drupal-8.1.x), le module est au statut "experimental". Les équipes de dev de Migrate et de Drupal core se sont concentrées dans un premier temps sur la migration complète depuis un D6 ou un D7, laissant les migrations custom un peu de côté dans un premier temps.

Même si tout n'était pas encore stabilisé dans Migrate, sur le projet ALPC nous avons pu l'utiliser pour mettre en place un import automatisé de contenu "communiqués de presse" à partir d'un flux XML.

Pour la vérité historique, j'ai moi même lamentablement échoué a faire marcher ce Migrate tout frais. Il aura fallu l'intervention héroïque d'un JYGastaud  plus déterminé que jamais, jonglant brillamment entre modules contribs et libraries en version de dev, pour que la synchro Migrate finisse par faire le boulot ;)

## 8 - Paragraphs & WYSIWYG

### 8.1 - WYSIWYG Sucks, paragraphs rocks !

Rien de véritablement spécifique à Drupal 8 ici, l'excellent module `paragraphs` existe en D7. Mais à ma connaissance, il a été jusqu'ici peu utilisé chez Clever Age.

Le principe étant de proposer une contribution structurée en paragraphes représentant des briques de contrib ayant leurs propres champs / layouts. Les paragraphes sont des entités embarquées au sein des nodes (ou d'autres entités), leur création / suppression / édition se fait directement au sein du formulaire d'édition de l'entité hôte.

La quasi totalité de la contribution sur ALPC se fait au travers de paragraphes, autorisant les contributeurs à créer et organiser des structures HTML complexes sans connaissance particulière ni recours au WYSIWYG.

Nous avons conservé un WYSIWYG minimaliste, pour faire du gras, de l'italique, des listes à puces et des tableaux. Pour toutes les autres structures, des plus simples (titre h2, texte) au plus complexes (carrousel, accordéon) les contributeurs ont recours à la création d'un paragraphe.

Nous avions envisagé en début de projet de ne pas mettre de WYSIWYG du tout, de se contenter d'un interpréteur Markdown, mais la réalité de la maquette, des contenus à contribuer et des profils de contributeurs parfois très peu a l'aise avec ce type d'outils nous ont amené à utiliser un CKEditor restreint.

### 8.2 - paragraphs D8, résolument moderne

Si *Paragraphs* existe en D7, notons toutefois que les équipes de dev ont fait un très beau travail d'intégration au modèle des entités dans la version D8. En effet paragraph D7 est un fork (assez ancien) d'un autre module du même type (`field_collection`) et souffre a cet égard d'une dette technique dans la modélisation de la relation entre le paragraphe et son contenu parent, qui rend difficile et parfois impossible, la traduction, la gestion des révisions, la migration ou l'intégration dans un workflow des contenus.

En D8, la relation hôte / paragraphe est modélisée par un champ EntityReference (en fait une version modifié du champ entityreference, entityreference_revision), ce qui en fait un type de relation intégrée de manière très homogène à la l'API Drupal.

## 9 - Style et organisation de code

### 9.1 - POO

Jusqu'à la version 7, Drupal était un framework essentiellement procédurale, on n'y trouvait quasiment pas de notions de programmation orientée objet. Par ailleurs, Drupal utilisait peu de libraries externes et n'avait donc pas de véritable gestion des dépendances.  
Drupal 8 s'intègre dans un écosystème PHP, tirant avantage de dépendances externes, à commencer par Symfony2.  
La quasi totalité des fonctions utilisées en Drupal 7 ont été restructurées et réécrites en POO.

Par exemple, obtenir le node courant en D7 :

```php
$node = menu_get_object('node');
```

En D8 :

```php
$node = \Drupal::request()->attributes->get('node');
```

Ou encore, pour déclencher le rendu d'un render array en D7 :

```php
$rendered_html = drupal_render($render_array);
```

En D8 :

```php
$rendered_html = \Drupal::service('renderer')->render($render_array);
```
La nature des entités (nodes, taxonomy term, users etc...) a également changé. En D7 les entités étaient déjà des objets mais non typés, il s'agissait de standard class, ne portant aucune méthode.

En D8 les entités de contenu comme les nodes, les block_content, les paragraphs etc... sont des instances de classes héritant toute d'une de `\Drupal\Core\Entity\ContentEntityBase`, et implémentant l'interface `\Drupal\Core\Entity\EntityInterface`. Ces classes exposent donc des méthodes permettant la manipulation des objets.  
Pour faire l'analogie avec les outils D7, on peut dire que les fonctions disponibles en utilisant `entity_metadata_wrapper`, sont maitenant disponibles directement dans un objet `$entity`.

Pour récupérer la valeur d'un champ d'un node en Drupal 7 :

```php
$items = field_get_item('node', $node, 'field_text');
$value = $items[0]['value'];
```
ou en utilisant `entity_metadata_wrapper` (module `entity`) :

```php
$wrapper = entity_metadata_wrapper('node', $node);
$value = $wrapper->field_text->value();
```

En Drupal 8, nativement :

```php
$value = $node->get('field_text')->first()->value()['value'];
```

### 9.2 - Où sont les hooks ?

En Drupal 7, tout est **hook**. Les hooks servent à la fois à déclarer des données (hook souvent suffixé par *_info*), à déclencher des actions un moment particulier et à altérer des données ou des structures HTML en cours de production (render arrays).  

En Drupal 8, un certain nombre de hooks ont été abandonnés au profit de méthodes plus usuelles en Symfony.

#### 9.2.1 - Annotations & fichiers Yaml

Les hooks "déclaratifs" ont presque tous disparu en D8. On utilise en remplacement soit des classes annotées ou des fichiers en configuration en .yml

En Drupal 7, un bloc était défini par :

```php

function module_block_info() {

  return array(
    'info' => 'alpc_logo_block',
  );
}

function module_block_view($delta) {

  if ('alpc_logo_block' === $delta) {

    $block = array(
      'subject' => t('Alpc Logo Block'),
      'content' => array(
        '#type' => 'link',
        '#text' => 'Accueil Région Aquitaine Limousin Poitou-Charentes',
        '#path' => '<front>',
      ),
    );
  }
}
```

En Drupal 8, déclaration d'un bloc en utilisant les annotations :

```php
<?php

namespace Drupal\alpc_common\Plugin\Block;

use Drupal\Core\Block\BlockBase;
use Drupal\Core\Url;

/**
 * Provides a 'Logo' block.
 *
 * @Block(
 *   id = "alpc_logo_block",
 *   admin_label = @Translation("ALPC logo block"),
 * )
 */
class AlpcLogoBlock extends BlockBase {

  /**
   * {@inheritdoc}
   */
  public function build() {
    return [
      '#type' => 'link',
      '#title' => 'Accueil Région Aquitaine Limousin Poitou-Charentes',
      '#url' => Url::fromRoute('<front>'),
    ];
  }

}
```
La présence de cette class définit un plugin block comme évoqué en 2.1.
Elle joue les rôles des hook_block_info et hook_block_view de Drupal 7. La déclaration passe par les annotations et le contenu du bloc est retourné par la methode build().

En Drupal 7, la déclaration d'une route se faisait par l'implémentation du hook_menu.

Exemple, en D7 :

```php

function module_menu() {

  return array(
    'custom-page' => array(
      'title' => t('Custom page'),
      'page callback' => '_module_custom_page',
		'access arguments' => array('access content'),
    ),
  );
}

function _module_custom_page() {

  return array(
    '#markup' => 'here the custom content',
  );
}
```

En Drupal 8, le hook_menu n'existe plus, il a été remplacé par un fichier .yml et les 'page callback' sont devenus des *controllers*. Un exemple tiré du projet ALPC, une route renvoyant un contenu destiné à être rendu dans une popin :

`alpc_common.routing.yml`

```yaml
alpc_common.subscribe_popin:
  path: '/popin/subscribe'
  defaults:
    _controller: '\Drupal\alpc_common\Controller\AlpcCommonController::alpcCommonSubscribePopin'
    _title: ''
  requirements:
    _permission: 'access content'
```

`AlpcCommonController.php`

```php
<?php

namespace Drupal\alpc_common\Controller;

use Drupal\Core\Controller\ControllerBase;

class AlpcCommonController extends ControllerBase{

  public function alpcCommonSubscribePopin() {

	$content = ... // construit le render array du contenu

	return $content;
  }
}
```

#### 9.2.2 - Evénements

S'agissant des hooks dont l'objet est de déclencher une action à un moment particulier, Symfony intègre un mécanisme permettant cela, il s'agit des événements. Les événements Symfony ne reposent pas sur une convention de nommage de fonction mais sur des services tagués.

Nous n'avons pas utilisé d'événements pour le projet ALPC, nous n'en avons pas eu besoin, peu de hooks ayant été abandonnés au profit du système d'événements. Certains problèmes de performance n'ont pas encore été résolus et par ailleurs, une part de la communauté des développeurs Drupal est encore réticente du fait de la verbosité du code nécessaire en comparaison des hooks Drupal.

Toutefois, il est probable qu'à l'avenir de plus en plus de hooks seront abandonnés ou dépréciés au profit d'événements.

### 9.3 - Services Symfony

Les services Symfony sont des classes déclarées dans des fichiers yaml. Une fois un service déclaré, il devient possible de l'appeler en utilisant le container de service. Dans l'exemple vu plus haut :

```php
$rendered_html = \Drupal::service('renderer')->render($render_array);
```

Le rendu d'un render array est assuré par la méthode `render` du service `renderer`.

La déclaration d'un service par un module se fait dans le fichier `nom_du_module.services.yml`. Un exemple sur ALPC :

```yaml
parameters:
  alpc_home.extrafields.conf: alpc_home.extrafields
services:
  alpc_home.xfield_manager:
    class: Drupal\alpc_home\Service\XfieldManager
    arguments: ['%alpc_home.extrafields.conf%']
```

Les services déclarés sont instanciés à l'initialisation d'une requête PHP, et tout ce dont ils ont besoin pour fonctionner leur est passé à ce moment là, c'est l'injection de dépendances. Il est possible d'injecter de la configuration ou encore d'autres services.

Les services Symfony permettent notamment d'organiser le code source Drupal de manière assez lisible. Lorsque que notre fichier .module commence à devenir trop gros (et même avant que cela arrive), il est préférable de séparer notre code dans des services, que nous appelons depuis notre fichier .module.  
C'est une alternative à l'usage des fichiers .inc présentant l'avantage de régler le problème de l'ordre de chargement des fichiers par l'injection de dépendances.  

Exemple d'appel au service `alpc_home.xfield_manager ` pour la déclaration et le rendu d'extrafields :

`alpc_home.module`

```php
<?php

/**
 * @file
 * ALPC home module file.
 */

use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Entity\Display\EntityViewDisplayInterface;
use Drupal\alpc_home\Service\XfieldManager;

function alpc_home_entity_extra_field_info() {

  /** @var XfieldManager $xfield_manager */
  $xfield_manager = \Drupal::service('alpc_home.xfield_manager');

  return $xfield_manager->getXfieldDefinitions();
}

function alpc_home_entity_view(
  array &$build,
  EntityInterface $entity,
  EntityViewDisplayInterface $display,
  $view_mode
) {

  \Drupal::service('alpc_home.xfield_manager')->processViewXfields(
    $build,
    $entity,
    $display
  );
}
```

On pourrait ainsi imaginer que le fichier .module d'un module D8 ne soit composé que d'implémentations de hooks qui ne font eux même qu'appeler des services.

### 9.4 - Structure d'un module

La structure d'un module Drupal a évolué :

```
module_name
|__ config
    |__ install
        *.yml
    |__ optionnal
        *.yml
|__ src
    |__ Service
        *.php
    |__ Controller
        *.php
    |__ Plugin
        |__ Block
            *.php
        |__ Field
            |__ Formatter
                *.php
|__ templates
   |__ *.html.twig
|__ module_name.info.yml
|__ module_name.module
|__ module_name.services.yml
|__ module_name.routing.yml
|__ module_name.libraries.yml
```

* Le fichier .info de Drupal 7 devient .info.yml : il s'agit donc d'un fichier descriptif en yaml. Il est cependant relativement semblable à la version D7 dans sa structure.

* Le fichier .module est en tout point ressemblant à sa version D7. Il s'agit d'un simple fichier PHP dans lequel on écrit des fonctions procédurales.

* Les fichiers .services|routing|libraries.yml sont des fichiers de configuration Symfony. Ils sont lus au runtime et leur contenu n'est jamais écrit en base de données.

* Le répertoire **config** contient des fichiers de configuration Drupal, ils sont lus et **écrits en base** lors de l'installation du module (install) ou lors de l'installation du module qui les contient ou du module dont ils dépendent (optionnal).

* Le répertoire **src** contient des classes. Les services et les controllers sont déclarés dans les fichiers de conf Symfony, les plugins (comme les blocks et les field formatters, mais il en existe d'autres) sont déclarés par annotation.

### 9.5 - Les libraries

En Drupal 8, les fichiers CSS ou JS sont inclus au travers de libraries. Il n'est pas possible d'inclure un fichier CSS ou JS directement, le fichier doit être déclaré dans le .libraries.yml du module comme un élément d'une library. C'est ensuite la library entière (qui peut contenir du CSS et du JS) qui est appelée dans le code, en utilisant la clé '#attached' d'un render array.

```php

return [
  '#theme' => 'image',
  '#uri' => 'public://img.jpg',
  '#alt' => '',
  '#attached' => [
    'library' => [
      'nom_library'
    ],    
  ],
];

```

### 9.6 - Structure d'un thème

```
nom_theme
|__ templates
  |__ *.html.twig
|__ nom_theme.info.yml
|__ nom_theme.libraries.yml
|__ nom_theme.theme
```

* Le fichier .info.yml est assez semblable au .info de D7.  

* Le fichier .libraries.yml est identique à ce que l'on a vu pour les modules. Notons toutefois que les libraries peuvent être inclues par un thème en utilisant une fonction Twig :

```twig
{{ attach_library('nom_module|theme/mon_library') }}
```
Ou encore être inclues sur toutes les pages du site, en les appelant dans le fichier nom_theme.info.yml :

```yaml

name: Alpc
type: theme
base theme: cleverdrop
description: 'Theme for ALPC project'
package: ALPC
version: 8.1.0
core: 8.x
libraries:
  - alpc/alpc-commons
```

* Le fichier nom_theme.theme est équivalent au fichier template.php que l'on trouvait dans les thèmes D7. Il s'agit d'un fichier PHP dans lequel on écrit des fonction procédurales. Toutefois, comme évoqué en 4.3, il est préférable de ne pas utiliser ce fichier. Les thèmes D8 n'ont pas vocation à contenir du PHP afin de faciliter leurs prises en main directes par les dev front.

## 10 - Le(s) cache(s) de Drupal 8

Cache système et cache de rendu

Cache rendu D7

  * Cache de page
  * Cache de block
  * Cache de rendu (optionnel)

Cache rendu d8

  * Internal page cache
  * Dynamic page cache
  * Render cache (obligatoire)

Système de cache externe

***En attente d'infos de Tristan***

___

En Drupal 7, s'agissant du rendu des pages, il existe 3 couches de cache :

  * Cache de page
  * Cache de block
  * Cache de rendu (optionnel)

Ces entrées de cache contiennent du code html prêt à être renvoyé directement.

Par ailleurs, il existe plusieurs données système mises en cache, mais elle ne sont pas directement impliquées dans le rendu des pages, elles ne contiennent pas de html. Les routes, les services, la liste des hooks implémentés par module, ou encore la liste des templates disponibles sont des données systeme mises en cache, elle ne sont mises à jour que lors d'une reconstruction volontaire du cache.
___


## 11 - Versioning & déploiement de configurations

### 11.1 - Configuration Drupal

Une des spécificités notable de Drupal, est que la configuration du site est stockée en base de donnée. Lorsque Drupal construit une page, il va lire la configuration en base et seulement en base (exception faite des fichiers de conf Symfony comme routing.yml ou services.yml). Celà permet une gestion de la configuration *à chaud* depuis le back-office, par des personnes qui ne sont pas developpeurs. C'est l'esprit même du systeme, qui même s'il tend à ce professionnaliser, reste dans sa conception un outils à destination aussi des webmasters.

Pour nous intégrateurs de solution, celà permet de livrer des sites qui pourront évoluer sans intervention de notre part, tant dans leur contenu que dans leur configuration.

Cependant, ce fonctionnement tend à complexifier les processus de livraison, puisque en plus de livrer du code, il nous faut aussi livrer de la configuration, qui se trouve en base de donnée.

### 11.2 - Features & Drupal 7

En Drupal 7, l'outils permettant celà est le module features.
Features lit en base de donnée les informations qui relevent de la configuration et propose d'en identifier certaines qui seront écrites dans le code d'un module, c'est **l'update**. Une fois le code du module déployé sur un site distant, si features est installé, on va pouvoir réaliser l'opération inverse, qui consiste à écrire en base de donnée des informations de configuration présentes dans le code, c'est le **revert**.

Même si le module features nous a permis pendant des année de déployer avec succès des configurations, il pose de nombreux problèmes.

* Il n'a pas été pensé pour ça. Features a pour vocation de permettre de packager du code et de la configuration dans un module en vue de sa réutilisation dans d'autres projets.

* Il n'existe pas d'API unifiée pour les données de configuration. La configuration en Drupal 7 peut prendre des formes tres différentes, entrée dans la table variables, données issue d'implémentation de hook déclaratifs (hook_xxx_info), model custom.

* Conséquemmet au point précédent, il existe de nombreuse méthodes différentes pour écrire de la conf dans le code (update) et pour faire le chemin inverse lors du déploiement (revert), dont certaines dépendent d'API externes (ctools par exemple). Le code généré par Features est donc complexe, hétérogène, et peu robuste.

### 11.3 - Configuration management & Drupal 8

Avec Drupal 8 est arrivé la notion de config management. Il s'agit d'une vaste remise à plat des concepts liés au stockage des configurations.

En Drupal 8, les configurations ne peuvent être que de 2 types :

  * Simple config : pour les structures simples et uniques
  * Config entities : pour les types structures plus élaborées, pouvant avoir plusieurs "instances".

Par exemple, les infos relatives au site (nom du site, email du site) sont stockées dans une simple config (system.site). Une vue est une instance du type de config entity "views.view" (ex. views.view.alpc_agenda)

Toutes peuvent être écrites dans le code sous la forme d'un fichier .yml
On peut ainsi écrire dans le code l'état de nos configuration en base, déployer le code et remonter la configuration sur un environnement différent. Ces mécanisme d'import / export sont natifs et peuvent être fait en ligne de commande.

Les configurations peuvent être écrites comme un ensemble unique, dans un seul répertoire, en tant "la config du site", ou peuvent être embarquées au sein d'un module, chaque module fournissant ainsi les éléments de configurations qui lui sont relatives. A noté que si le config management est capable de gérer les dépendances entre les configuration d'un même lot, ce n'est pas le cas pour des configurations qui vivent séparement dans des modules.



### 11.4 - Features 8.x & ALPC :(

Au lancement des developpements d'ALPC