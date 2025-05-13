Écriture d’un code pour la pile EtherCAT Master IgH
======================================================

Objectif : écrire un code basé sur la pile EtherCAT Master IgH afin de réaliser des **mesures temporelles dans un système en temps réel strict**.

1. Mise en contexte
--------------------

EtherCAT utilise la technologie d'**horloge distribuée (Distributed Clock)** pour synchroniser tous les esclaves. Cela permet de garantir des **performances en temps réel** et un **contrôle précis** du système.

Horloges distribuées – Généralités
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Dans les systèmes distribués, plusieurs entités distantes doivent agir simultanément. Il est donc crucial de synchroniser précisément les équipements.  
La méthode la plus performante connue à ce jour pour les réseaux distribués consiste à **aligner les horloges du système**, comme décrit dans le **Precision Time Protocol (PTP)**.

.. note::

   Le PTP (norme IEEE-1588) est utilisé pour synchroniser les horloges sur un réseau (ex: Ethernet) via l’échange d’horodatages.

Le PTP suit un modèle **hiérarchique maître-esclave**, où le maître distribue son horloge aux esclaves.

Contrairement aux systèmes dits *synchrones*, le PTP reste robuste même en présence de délais ou d’erreurs de communication. Cela apporte une meilleure **tolérance temporelle**.

Horloges distribuées dans EtherCAT
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Dans EtherCAT, la synchronisation des horloges peut aussi s'étendre à des réseaux industriels entiers, par exemple dans une usine, grâce au protocole **IEEE-1558**.

Cette synchronisation permet :
- des mesures précises de transfert ou d’acquisition,
- des mouvements **coordonnés** entre plusieurs axes servo.

.. important::

   La synchronisation est entièrement **matérielle**.

Le **sous-appareil DC** maître distribue le temps aux autres à chaque cycle.  
Comme le signal subit un délai de propagation, celui-ci doit être **mesuré et compensé**. Cela se fait soit au **démarrage**, soit **en continu**.

Grâce à la **topologie en anneau** d’EtherCAT, les délais de propagation sont faciles à mesurer par le maître (et les esclaves peuvent faire de même).

2. Écriture du code
--------------------

Étude du code `dc_rtai.c`
^^^^^^^^^^^^^^^^^^^^^^^^^

Le fichier `dc_rtai.c` est un exemple fourni avec la pile **IgH EtherCAT Master**.

Contrairement aux autres exemples comme `dc_user.c` ou `user.c`, il est conçu pour des systèmes **temps réel strict (hard real-time)** grâce à :

- l'utilisation de **RTAI (Real-Time Application Interface)**,
- un noyau Linux modifié (RTAI, Xenomai, PREEMPT-RT…).

.. warning::

   `dc_rtai.c` **nécessite un OS temps réel**. Il ne peut pas tourner sur un noyau Linux standard.

Objectif : écrire notre propre code
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Nous allons écrire notre propre programme en nous basant sur la structure de `dc_rtai.c`, pour effectuer **des mesures temporelles** dans un contexte **temps réel strict**.

Dans `dc_user.c`, on mesure déjà :
- la **période**,
- le **temps d’exécution**,
- la **latence**.

Notre but est de reproduire ces mesures mais cette fois en garantissant un comportement *hard real-time*.

1re étape : adaptation aux esclaves de la maquette
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Nous avons commencé par adapter `dc_user.c` à notre configuration matérielle (maquette avec Raspberry Pi).

- Deux esclaves du code initial ont été conservés.
- Le **compteur IDS** a été remplacé par un **Beckhoff_EL5101** (également un compteur).

Nous prévoyons d’adapter **`dc_rtai.c` de la même façon**, en ajustant le fichier à la topologie réelle de notre banc d’essai.

---

