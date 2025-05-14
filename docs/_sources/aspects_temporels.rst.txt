Aspects temporels
====================

Bien que le timing EtherCAT soit hautement déterministe et que par conséquent les problèmes de timing soient rares, il y a quelques aspects qui peuvent (et doivent) être traités.

1. Profilage de l’interface de programmation applicative
-----------------------------------------------------------

Un des aspects de timing les plus important est le temps d’exécution des fonctions de l’API, qui sont appelées dans un contexte cyclique. Ces fonctions prennent une part importante du timing d’ensemble de l’application. Pour mesurer le timing de ces fonctions, le code suivant a été utilisé :

.. code-block:: c

    c0 = get_cycles();
    ecrt_master_receive(master);
    c1 = get_cycles();
    ecrt_domain_process(domain1);
    c2 = get_cycles();
    ecrt_master_run(master);
    c3 = get_cycles();
    ecrt_master_send(master);
    c4 = get_cycles();

Entre chaque appel d’une fonction de l’API, le compteur d’horodatage d’estampille du microprocesseur est lu. Les différences des compteurs sont converties en µs au moyen de la variable ``cpu_khz``, qui contient le nombre d’incréments par ms.

Pour la mesure réelle, un système avec un microprocesseur à 2.0 GHz a été utilisé pour exécuter le code ci-dessus dans un fil d’exécution RTAI avec une période de 100 µs.  
La mesure a été répétée *n = 100* fois et les résultats ont été moyennés. Ils sont visibles dans le tableau 8.1.

.. list-table:: **Tableau 8.1 – Profilage d’un cycle d’application sur un processeur à 2.0 GHz**
   :header-rows: 1
   :widths: 30 30 30

   * - Élément
     - Durée moyenne [µs]
     - Écart-type [µs]
   * - ecrt_master_receive()
     - 8.04
     - 0.48
   * - ecrt_domain_process()
     - 0.14
     - 0.03
   * - ecrt_master_run()
     - 0.29
     - 0.04
   * - ecrt_master_send()
     - 2.18
     - 0.17
   * - **Cycle complet**
     - **10.65**
     - **0.69**

Les fonctions qui opèrent uniquement sur les **structures de données internes** du maître, comme ``ecrt_domain_process()`` ou ``ecrt_master_run()``, sont généralement **très rapides** (inférieures à 1 µs).  
Cela s’explique par le fait qu’elles n’ont **pas besoin d’accéder au réseau**, **n’attendent aucune trame**, et **n’interagissent pas avec du matériel externe**.

Parmi elles, ``ecrt_domain_process()`` montre une **déviation standard très faible**, ce qui indique une **grande efficacité** et **une exécution régulière**, quasiment constante à chaque cycle.  
En revanche, ``ecrt_master_run()``, qui traite la **logique interne basée sur les données reçues**, présente plus de **variabilité temporelle**, car elle dépend fortement **des conditions d’exécution et de l’état actuel du système**.

À l’opposé, les fonctions qui interagissent directement avec le matériel – comme ``ecrt_master_receive()`` et ``ecrt_master_send()`` – prennent **plus de temps** à s'exécuter.

- ``ecrt_master_receive()`` est la **plus lente**. Elle gère une **interruption Ethernet (ISR)** dès l’arrivée des données, analyse les datagrammes reçus, puis copie leur contenu en mémoire.
- ``ecrt_master_send()``, elle, se contente de **construire et envoyer une trame**. Elle est donc environ **quatre fois plus rapide** que la fonction de réception.

.. note::

   Ces différences sont visibles dans le tableau de profilage des fonctions, et montrent l’impact significatif du traitement matériel sur le timing global.


**Limites de la fréquence théorique**:

La fréquence théorique maximale d’exécution des cycles peut atteindre **jusqu’à 100 kHz**, mais elle est limitée en pratique par deux facteurs :

1. Le processeur doit continuer à exécuter le **système d’exploitation** entre les cycles temps réel.
2. Les **trames EtherCAT** doivent être entièrement **envoyées et reçues** avant le prochain cycle.

Ces contraintes réduisent la fréquence réelle exploitable, et doivent être prises en compte dans toute évaluation de performances temps réel.

2. Mesure des cycles du bus
-----------------------------

La durée pendant laquelle une trame EtherCAT circule "sur le câble" est un élément fondamental à mesurer pour évaluer les limites de fréquence d’un système temps réel.

Pour obtenir cette mesure, il faut déterminer précisément **deux instants clés** :

1. **Le début de la transmission** : moment où le matériel Ethernet commence à envoyer la trame.
2. **La fin de la réception** : moment où la trame est totalement reçue par le matériel à l’autre bout.

Cependant, ces instants sont **difficiles à capter avec précision** pour plusieurs raisons :

- Les **interruptions système** sont parfois désactivées pour garantir le déterminisme, ce qui empêche une notification immédiate de l’événement au CPU.
- Même avec les interruptions activées, il subsiste un **délai inconnu** entre l’événement réel (trame envoyée/reçue) et le traitement de l’interruption.
- Le **polling** (lecture cyclique d’un registre matériel) serait trop lent ou introduirait un biais dans la mesure.
- Les événements sont **difficilement repérables** par logiciel sans instrumentation matérielle.

.. tip::

  **Solution recommandée** : utiliser une **mesure électrique** avec des outils comme :
   - un oscilloscope numérique,
   - une sonde logique,
   - ou un analyseur réseau (hardware).

Ces outils permettent d’obtenir des mesures fiables des temps de transit des trames Ethernet dans le réseau.

Importance du cycle du bus
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Dans une application temps réel EtherCAT, la **durée du cycle du bus** détermine directement la **fréquence maximale** des échanges.  
Si cette fréquence est trop élevée, les trames envoyées à la fin d’un cycle peuvent ne pas être totalement reçues lorsque le cycle suivant commence.

Dans ce cas :
- La fonction ``ecrt_domain_process()`` détecte que les **nouvelles données ne sont pas arrivées** en vérifiant le **compteur de datagrammes**, qui reste inchangé.
- Elle **conserve alors les anciennes données** pour éviter d’utiliser des valeurs incomplètes ou corrompues.
- Le **maître EtherCAT** remet ces datagrammes dans la **file d’envoi** et génère un **avertissement** indiquant que des paquets ont été ignorés.

Conclusion et seuils pratiques
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Sur un système à **2.0 GHz**, on atteint typiquement une fréquence de **25 kHz sans perte de trames**.

Cependant, cette fréquence maximale peut être améliorée en :
- **Utilisant un matériel réseau plus rapide** (éviter les chipsets comme RealTek, préférer Intel ou compatibles industriels).
- **Optimisant le traitement des interruptions** avec un **ISR dédié à EtherCAT**, pour réduire la latence de réponse matérielle.