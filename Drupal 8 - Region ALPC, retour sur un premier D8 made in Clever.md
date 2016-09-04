# Drupal 8 - Region ALPC, retour sur un premier D8 made in Clever

Petit tour d'horizon des nouveautés D8 (et de leur petits pièges) qui se sont trouvées sur notre route lors de la réalisation de [http://www.laregion-alpc.fr](http://www.laregion-alpc.fr)

## 1 - Symfony2 & POO 

Drupal 8 utilise beaucoup de composants Symfony. Pour autant ce n'est pas une application Symfony full stack (comme EZ publish 5/6 par exemple). Un certains nombre de concepts (notament dans la modélisation des contenus) n'ont pas été "symfonyfiés" (ce mot existe. Si, c'est vrai.)  

Cependant, la totalité de la base de code du core (et de la grande majorité des modules contrib portés et stables en D8) a été repensée / réécrite en POO.  

Pour developper en Drupal 8, sans être obligé de savoir vraiment faire du symfony, il est tout de même préférable de maîtriser les notions de :

  * Injection de dépendance et services symfony
  * Structures et syntaxe des fichiers de conf en .yml
  * Annotations
  * Syntaxe TWIG

Plus généralement, il faut avoir les bases de la POO php :

  * Classes abstraite et héritage
  * Interface
  * Methode static / protected / public

Même si ça à l'air d'un gros morceau a avaler pour qui n'aurait fait que du drupal 6/7 jusque là (et donc quasiment pas de POO), ça n'a rien d'insurmontable. Ça n'a en tout cas pas posé de problème particulier sur ALPC, parmis les devs il y avait 1 Symfoniste confirmé et 2 drupalistes.

## 2 - Les blocks

### 2.1 - Plugin baby !

En Drupal 8, la notion de block a pas mal évolué en Drupal 8, tant dans les concepts que dans le code.

En D7 un block est un fragment de contenu, déclaré dans un hook et disponible pour être placé dans une region du thème. Un block donné ne peut apparaître qu'en un seul exemplaire, et dans une seule région.
Ce mécanisme limité peut être étendu en utilisant des modules contrib comme context, panel ou display suite.

Drupal 8 a intégré la souplesse de gestion des blocks apportée par les modules cités plus haut dans le core en utilisant le duo plugin + config entity.

Le plugin est d'abord définit dans le code, puis une fois le plugin block disponible dans l'interface, on peut le "placer" dans une région pour un thème donné, surcharger son titre, et ajuster les settings (visibilité par exemple). C'est a ce moment la qu'une instance de config entity est crée en base. La config entity contient toutes les infos relatives au placement du plugin block.

On peut créer plusieurs config entities pour un même plugin, avec des settings différents à chaque fois.

### 2.2 - Custom block (block_content)

En Drupal 7 il n'est pas possible nativement de créer des blocks configurables auxquels on attache des champs. C'est un besoin assez courant auquel repondent les modules contrib `nodeblock` et plus récemment (et plus proprement) `bean`.  

Là encore drupal 8 a intégré cette problématique dans le core. Il s'agit quasiment d'un port du module `bean` sous le nom de *Custom Block* (nom machine `block_content`).

le module *Custom Block* definit le type d'entité `block_content`. Il est possible de créer des types de blocks (analogues aux type de contenus) auxquels on attache des champs pour la contrib. Ensuite, chaque `block_content` créé donne lieu à la création d'un plugin block. Il est donc possible de placer un block contribué comme un block definit dans le code.

Par exemple sur ALPC, les blocks image cliquables visibles dans le menu principal, dans la sidebar des articles et au bas de la page d'accueils sont des `block_content`.

## 3 - Form modes

Le form mode est l'équivalent du view_mode côté formulaire. Autrement dit il s'agit d'une structure qui stock, pour un bundle donné (type de cotenu, type de block etc...), la visibilité, l'ordre d'apparition et le widget de chaque champs.  

Il devient donc possible de créer plusieurs visualisations possible d'un même formulaire d'entité.  

Su ALPC nous l'avons peu utilisé, le seul exemple est le formulaire d'inscription à la newsletter qui apparait sous une forme différente en homepage et sur les articles. 

## 4 - {{ TWIG }}

### 4.1 - Render arrays & logique d'assemblage
 
D8 integre TWIG en lieu et place de phpTemplate. Et c'est bien.  
Le twig embarqué dans drupal est un twig etendu, enrichi de nombreuses fonctions dédiés.
Contrairement à phpTemplate, twig est intimement lié aux mécanismes de rendu de drupal. Concretement celà signifie que twig peut non seulement printer des chaînes, comme il le fait naturellement au sein de n'importe quel framework, mais il est également capable d'invoquer, de manière transparente la render API de drupal. Autrement dit, TWIG peut printer des render arrays.

Par exemple dans un template de node comme :

```twig
<article>
  <h1>{{ title }}</h1>
  {{ content }}
</article>
```

`title` contient une chaîne, et `content` contient un render array (incluant tous les champs) produit par le view_mode du node.

Le fait que twig sache gérer les render array implique qu'une grande partie de la logique d'assemblage des templates se fait dans les render arrays. Ce n'est pas (ou rarement) au sein d'un template que l'on va appeler un sous template avec un include.  

Dans des cas d'assemblage 1 template parent / multiple templates enfants, comme typiquement le template de node vu plus haut, le template parent "reçoit" un render array contenant des enfants (les champs du node par exemple), et le print tout simplement. La render API reprend alors la main et fait appel aux templates des champs pour resoudre le render array.

En bref, twig ne porte pas la logique d'assemblage contrairement à l'usage qui peut en être fait dans d'autres frameworks.

Les avantages sont mulitples : 

  * Simplicité des templates : ils sont nombreux (si peu mutualisés) mais très atomique, donc courts et lisibles.
  * Maintient complet de la logique de suggestion de templates, chaque template de chaque element aussi petit soit il a son propre template, lequel template peut être choisis en fonction du contexte parmis un ensemble par convention de nommage. Ce qui rend le systeme très flexible.
  * Si la logique d'assemblage n'est pas écrite en dur dans les templates, il devient possible d'exposer à la contribution des mécanisme de layout avancé (view_modes, panels, paragraphs etc ...)

Exemple d'un enssemble de template pour le rendu d'un node dans le projet ALPC :

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

### 4.2 - Attributes

Le twig de drupal inclut également la gestion d'une objet de type `Attributes`. Il s'agit d'une structure portant des attributs html. Cette objet expose de multiples méthodes permettant de manipuler des attributs à la volée au moment de les printer.

Par exemple, le template des titres secondaires dans les articles d'ALPC :

```twig
<section class="second-title">
  {% set classes = ['title-outerline-blue', 'font-title'] %}
  <h2{{ attributes.addClass(classes) }}>{{ value }}</h2>
</section>
```

En Drupal 7, ce type de manipulation devait se faire dans un preprocess en php.

### 4.3 - Laissons les dev front travailler et les contributeurs contribuer !

D'une manière générale l'intégration de twig et son imbrication dans la render API de Drupal permet une grande souplesse de contribution, y compris s'agissant du layout des éléments, et ce sans franchir la frontière (interdite) de la contribution de html et css en back-office. Il devient possible (plus encore qu'en D7) de déléguer aux contributeurs la construction du puzzle et aux dev front la responsabilité du design des pièces.

Par ailleurs, le theme drupal peut désormais être entièrement géré par les dev front, sans connaissances particuliere de Drupal ou même de PHP.  
La ou en D7, une tres large partie du code php custom d'un projet se trouvait dans le theme, en D8 le thème peut (doit) être consistué exclusivement de Templates Twig, de feuilles de style, de fichiers js et d'assets (images, svg, json etc...)

Amis intégrateurs, les seules prérequis sont :

  * Accepter que la logique d'assemblage n'est pas (ou peu) dans les templates.
  * Connaitre les grands principes de cette logique d'assemblage (tres atomisé).
  * Connaitre les quelques extensions TWIG spécifiques à Drupal.

Sur ALPC, pour des raisons de planing principalement, nous n'avons pas pu mettre en oeuvre ce principe, ce sont les dev Drupal qui se sont chargé d'écrire les templates à partir d'une intégration statique réalisée en amont. Par ailleurs, le thème alpc embarque malheureusement quelque lignes de code php.  
Mais la prochaine fois c'est la bonne :)

## 5 - Composer

### 5.1 - Composer dans le core de D8 !

Parmis les nouveautés D8, on trouve l'integration de composer. On est encore au début de l'histoire, mais c'est prometeur.
En réalité c'est surtout le core de Drupal 8 qui utilise composer pour ses dépendances, mais lorsque l'on télécharge le core de drupal, on récupère un ensemble de sources, incluant les dépendances du core dans un repertoire vendor.

Toutefois, il est possible d'ajouter des dépendances dans le composer.json prévu à cet effet. Sur ALPC, c'est ce que nous avons fait, pour les libs elasticsearch et pour les libs des API des reseaux sociaux.

### 5.2 - Bien mais pas top

Mais celà a pose un problème dans les procedures de mise à jour. En effet, lors d'une mise a jour du core (sans composer) on recupere toutes les sources, y compris le repertoire vendor qui ne contient pas nos dependances ajoutées, il faut donc faire un backup et merger tout ça à la main, pas tres pratique.

Pour ALPC, vu le faible nombre de dependances spécifiques du projet, on a fait avec : utilisé composer sur les postes de dev uniquement et versionné tous les vendor.  
Lors des mise a jour du core, on a fait attention (pour la vérité historique, j'ai d'abord tout cassé sans rien comprendre, puis j'ai refléchis, et ensuite j'ai fais gaffe).

### 5.3 - drupal-project

Pour les prochaines fois, une solution complète est proposée par ce projet : 
[https://github.com/drupal-composer/drupal-project](https://github.com/drupal-composer/drupal-project)  
Dans cette approche full composer, toutes les dependances du projet (incluant le core de drupal et les modules contrib) sont gérées par composer, ce qui permet de ne versionner que le nécessaire et d'automatiser les mise à jours et les déploiements.

Il y aura peut être un article KB sur cet outils prochainement.

## 6 - Contact form

### 6.1 - Des formulaires d'entité

D8 introduit un nouveau modèle de formulaires de contact. Les contact form de D8 sont contribuables, il est possible de définir les champs du formulaire depuis le BO.

Dans l'histoire de Drupal (depuis D7 en tout cas), 2 approches différentes ont existé pour créer des formulaires en BO.

* Le modele "Webform", qui fournit des types de champs pour sa propre API, gère la construction du formulaire, son rendu en front, sa logique de validation et de soumission ainsi que la persistance des données soumise de manière spécifique.
* Le modele "EntityForm", qui s'appuit sur les concepts d'entité et de champs existant dans le core de Drupal. En effet, les formulaires utilisés pour la contribution des entités sont finalement des formulaire dont on gère la structure en back-office. C'est encore plus vrai avec l'arrivé des form modes de D8.

Contact Form est un module du core de Drupal 8 qui s'inscrit dans la continuité de EntityForm, les formulaires de contact sont en fait des entités dont on expose le formulaire d'édition en front.

Cette approche permet une utilisation totale des concepts liés aux champs des entités (notions de widget, validateurs, rendu, form modes, theming, stockage en base etc ...)

A noter toutefois que la persitance des données en base n'est pas (encore) native, par défaut, les formulaires ne font qu'envoyer un mail. Toutefois, le module contrib `contact_storage` complète `contact_form` pour disposer de véritables entités persistantes.

Sur ALPC nous avons utilisé des contact form pour tous les formulaires exposés en front. Le formulaire de contact biensûr, mais également le formulaire d'inscription à la newsletter, le formulaire d'inscription aux notification d'agenda et communiqués de presse, un formulaire de recherche rapide et le formulaire de dépôt d'offres d'emploi.

Ces formulaires sont très différents, et pour certains assez ambitieux niveau front / UX :
  
  * Soumission / Validation AJAX
  * Visibilité des champs modifiés à la volée
  * Autocomplete branché sur elasticsearch
  * Affichage en popin
  * Intégration HTML / CSS 100% custom

Dans l'ensemble, contact form nous a permis de faire tout ça, en écrivant un peu de code, mais toujours en exploitant les API natives de drupal, pas de js ou de php custom qui réinvente la roue.

### 6.2 - Too much

Contact form en fait même un peu trop au bout du compte. En effet le module a vraiment été conçu pour des formulaires de contact, et il existe donc par défaut des champs qui ne sont pas utiles dans le cadre des formulaires d'un autre type. Ce n'est pas tres grave, on peut utiliser les form modes pour choisir les champs que l'on souhaite exploiter.  

Plus gênant en revanche, la soumission d'un formulaire de contact **doit** envoyer un mail, c'est en dur dans le core, et de surcroit envoyer un mail à une adresse dont la saisie est obligatoire lors de la définition du formulaire.  
Dans le cas d'ALPC le seul formulaires dont on avait besoin qu'il envoit effectivement des mails (le formulaire de contact du site) devait exposer un champ dans lequel le visiteur pourrait choisir un service de la region, comme destinataire de son message. Impossible donc d'exploiter le mécanisme natif d'envoi de mail.  
Il a donc fallu empecher le mail de partir, et réimplementer une logique d'envois de mail parallèlement.  
(A cette occasion, nous avons essayé de nous appuyer sur `rules` pour le déclenchement de l'envois du mail, mais sans succès, `rules` est encore bien trop instable)

### 6.3 - Webform ? Yamlform ? Eform ?

Avec le recul, ce genre de petits soucis me font penser que `contact_form` devra évoluer vers plus de souplesse à l'avenir et integrer nativement `contact_storage` pour que son usage devienne aussi courant que l'était webform en D6 et D7.

Une autre alternative si `contact_form` n'étend pas son périmètre, serait d'utiliser le module `eform` (le nom du projet `entityform` porté en D8), dont la vocation est de couvrir complètement le besoin des formulaires contribués.

A noter enfin que webform ne sera probablement pas porté en D8, mais qu'il existe le projet `yamlform`, semblable dans l'approche (n'utilisant pas les entités et leur champs, mais sa propre api).

## 7 - Migrate

Le module `migrate` est dans le core de Drupal !  
Et c'est une bonne nouvelle. Cependant, à l'heure actuelle (drupal-8.1.x), le module est au statut "experimental". Les équipes de dev de migrate et de drupal core se sont concentrés dans un premier temps sur la migration complète depuis un D6 ou un D7, laissant les migrations custom un peu de côté dans un premier temps.

Même si tout n'était pas encore stabilisé dans migrate, sur le projet ALPC nous avons pu l'utiliser pour mettre en place un import automatisé de contenu "communiqués de presse" à partir d'un flux XML.

Pour la vérité historique, j'ai moi même lamantablement échoué a faire marcher ce migrate tout frais, et il aura fallu l'intervention héroïque d'un JYGastaud  plus déterminé que jamais, jonglant brillamment entre modules contribs et libraries en version de dev, pour que la synchro migrate finisse par faire le boulot ;)

## 8 - Paragraphs & WYSIWYG

### 8.1 - WYSIWYG Sucks, paragraphs rocks !

Rien de véritablement spécifique à Drupal 8 ici, l'excelent module `paragraphs` existe en D7. Mais a ma connaissance, il a été jusqu'ici peu utilisé chez Clever.

Le principe étant de proposer une contribution structurée en paragraphs représentant des briques de contrib ayant leur propre champs / layouts. Les paragraphs sont des entités embarqués au sein des nodes (ou d'autres entités), leur création / suppression / edition se fait directement au sein du formulaire d'edition de l'entité hôte.

La quasi totalité de la contribution sur ALPC se fait au travers de paragraphs, autorisant les contributeurs à créer et organiser des structures HTML complexes sans connaissance particuliere ni recours au WYSIWYG.

Nous avons conservé un WYSIWYG minimaliste, pour faire du gras, de l'italique, des listes à puces et des tableaux. Pour toutes les autres structures, des plus simples (titre h2, texte) au plus complexes (carrousel, accordeon) les contributeurs ont recours à la cration d'un paragraphe.

Nous avions envisagé en début de projet de ne pas mettre de WYSIWYG du tout, de se contenter d'un interpreteur Markdown, mais la réalité de la maquette, des contenus a contribuer et des profiles de contributeurs parfois tres peu a l'aise avec ce type d'outils nous ont amené à utiliser un CKEditor restreint.

### 8.2 - paragraphs D8, résolument moderne

Si paragraphs existe en D7, notons toutefois que les équipes de dev ont fait un tres beau travail d'intégration au modèle des entités dans la version D8. En effet paragraph D7 est un fork (assez ancien) d'un autre module du même type (`field_collection`) et souffre a cet égard d'une dette technique dans la modélisation de la relation entre le paragraph et son contenu parent, qui rend difficile et parfois impossible, la traduction, la gestion des revisions, la migration ou l'integration dans un workflow des contenus.

En D8, la relation hote / paragraph est modélisée par un champ entityreference (en fait une version modifié du champ entityreference, entityreference_revision), ce qui en fait un type de relation intégrée de manière tres homogène à la l'API Drupal.

## 9 - Style et organisation de code

### 9.1 - POO

Jusqu'à la version 7, Drupal était un framework essentiellement procédurale, on n'y trouvait quasiment pas de notions de programation orientée objet. Par ailleurs, drupal utilisait peu de libraries externes et n'avait donc pas de véritable gestion des dependances.  
Drupal 8 s'intègre dans un écosysteme php, tirant avantage de dépendances externes, à commencer par Symfony2.  
La quasi totalités des fonctions utilisées en drupal 7 ont été restructurée et réécrite en POO.

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
La nature des entités (nodes, taxonomy term, users etc...) a également changé. En D7 les entités étaient déjà des objects mais non typés, il s'agissait de standard class, ne portant aucune méthodes.

En D8 les entités de contenu comme les nodes, les block_content, les paragraphs etc... sont des instances de classes héritant toute d'une de `\Drupal\Core\Entity\ContentEntityBase`, et implémentant l'interface `\Drupal\Core\Entity\EntityInterface`. Ces classes exposent donc des méthodes permettant la manipulation des objets.  
Pour faire l'analogie avec les outils D7, on peut dire que les fonctions disponible en utilisant `entity_metadata_wrapper`, sont maitenant disponible directement dans un objet `$entity`.

Pour récupérer la valuer d'un champ d'un node en Drupal 7 : 

```php
$items = field_get_item('node', $node, 'field_text');
$value = $items[0]['value'];
```
ou en utilisant `entity_metadata_wrapper` (module `entity`) :

```php
$wrapper = entity_metadata_wrapper('node', $node);
$value = $wrapper->field_text->value();
```

En drupal 8, nativement :

```php
$value = $node->get('field_text')->first()->value()['value'];
```

### 9.2 - Où sont les hooks ?

En Drupal 7, tout est **hook**. Les hooks servent à la fois à déclarer des données (hook souvent suffix par info), à déclencher des actions un moment particulier, et à altérer des données ou des structures html en cours de production (render arrays).  

En Drupal 8, un certain nombre de hook ont été abandonnés au profit de méthodes plus usuelles en Symfony.

#### 9.2.1 - Annotations & fichiers Yaml

Les hook "déclaratifs" ou presque tous disparu en D8, on utilise en remplacement soit des classes annotées ou des fichiers en configuration en .yml

En drupal 7, un block était définit par :

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

En drupal 8, déclaration d'un block en utilisant les annotations :

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
Elle joue les rôles des hook_block_info et hook_block_view de Drupal 7. La déclaration passe par les annotations et le contenu du block est retourné par la methode build().

En drupal 7, la déclaration d'une route se faisait par l'implémentation du hook_menu.

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

En Drupal 8, le hook_menu n'existe plus, il a été remplacé par un fichier .yml et les 'page callback' sont devenus controllers. Un exemple tiré du projet alpc, une route renvoyant un contenu destiné à être rendu dans une popin :

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

#### 9.2.2 - Evenements

S'agissant des hooks dont l'objet est de déclencher une action à un moment particulier, Symfony integre un mécanisme permettant celà, il s'agit des evenements. Les evenements symfony ne repose pas sur une convention de nommage de fonction, mais sur des services tagés.

Nous n'avons pas utilisé d'évènements pour le projet ALPC, nous n'en avons pas eu besoin, peu de hook ayant été abandonnés au profit du système d'évènements. Certains problèmes de performance n'ont pas encore été résolu et par ailleurs, une part de la communauté des developpeurs Drupal est encore réticente du fait de la verbosité du code nécessaire en comparaison des hooks drupal.

Toutefois, il est probable qu'à l'avenir de plus en plus de hook seront abandonnés ou dépreciés au profit d'evenements.

### 9.3 - Services symfony

Les services symfony sont des classes, déclarées dans des fichiers yaml. Une fois un service déclaré, il devient possible de l'appeler en utilisant le container de service. Dans l'exemple vu plus haut :

```php
$rendered_html = \Drupal::service('renderer')->render($render_array);
```

Le rendu d'un render array est assuré par la methode `render` du service `renderer`.

La déclaration d'un service par un module se fait dans le fichier `nom_du_module.services.yml`. Un exemple sur alpc :

```yaml
parameters:
  alpc_home.extrafields.conf: alpc_home.extrafields
services:
  alpc_home.xfield_manager:
    class: Drupal\alpc_home\Service\XfieldManager
    arguments: ['%alpc_home.extrafields.conf%']
```

Les services déclarés sont instanciés à l'initialisation d'une requête php, et tout ce dont ils ont besoin pour fonctionner leur est passé à ce moment là, c'est l'injection de dépendance. Il est possible d'injecter de la configuration ou en core d'autres services.

Les services symfony permettent notament d'organiser le code source Drupal de manière assez lisible. Lorsque que notre fichier .module commence à devenir trop gros (et même avant que ça n'arrive), il est préférable de séparer notre code dans des services, que nous appelons depuis notre fichier .module.  
C'est une alternative à l'usage des fichiers .inc présentant l'avantage de regler le problème de l'ordre de chargement des fichier par l'injection de dependance.  

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

La structure d'un module Drupal a évoulé :

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

* Le fichier .info de Drupal 7 devient .info.yml il s'agit donc d'un fichier descriptif en yml. Il est cependant relativement semblable à la version D7 dans sa structure.

* Le fichier .module est en tout point ressemblant à sa version D7. Il s'agit d'un simple fichier php, dans lequel on écrit des fonctions procédurales.

* Les fichiers .services|routing|libraries.yml sont des fichiers de configuration Symfony, ils sont lus au runtime et leur contenu n'est jamais écrit en base de donnée.

* Le repertoire **config** contient des fichiers de configuration Drupal, ils sont lus et **écrits en base** lors de l'installation du module (install) ou lors de l'installation du module qui les contient ou du module dont ils dépendent (optionnal).

* Le repertoire **src** contient des classes. Les services et les controllers sont déclarés dans les fichiers de conf Symfony, les plugins (comme les blocks et les field formatters, mais il en existe d'autres) sont déclarés par annotation.

### 9.5 - Les libraries

En Drupal 8, les fichiers css ou js sont inclus au travers de libraries. Il n'est pas possible d'inclure un fichier css ou js directement, le fichier doit être déclaré dans le .libraries.yml du module comme un element d'un librarie, et c'est ensuite la library entière (qui peut contenir de la css et du js) qui est appelée dans le code, en utilisant la clée '#attached' d'un render array.

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

### 9.6 - Sructure d'un thème

```
nom_theme
|__ templates
  |__ *.html.twig
|__ nom_theme.info.yml
|__ nom_theme.libraries.yml
|__ nom_theme.theme
```

* Le fichier .info.yml est assez semblable au .info de D7.  

* Le fichier .libraries.yml est identique à ce que l'on a vu pour les modules, notons touteois que les libraries peuvent être inclues par un theme en utilisant une fonction twig : 

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

* Le fichier nom_theme.theme est équivalent au fichier template.php que l'on trouvait dans les themes D7. Il s'agit d'un fichier php dans lequel on écrit des fonction procédurales. Toutefois, comme évoqué en 4.3, il est préférable de ne pas utiliser ce fichier, les themes D8 n'ont pas vocation à contenir du php, ce pour faciliter leur prise en main direct par les dev front.

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

### 11.3 - CMI & Drupal 8

### 11.4 - Features 8.x & ALPC :(