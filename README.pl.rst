
Ostatnio AWS opublikowało szablony [1]_ CloudFormation, aby uruchomić prywatną sieć testową Ethereum.
Ich rozwiązanie zawiera aplikację do przeglądania bloków, majnera itp. Tak wielkie, jak to jest, to rozwiązanie nie jest łatwe do zastosowania do uruchomienia node'a na mainnecie.

Po pierwsze, spróbujmy odpowiedzieć, dlaczego ktokolwiek potrzebuje prywatnie hostowanego node'a na mainnet Ethereum zamiast polegać na Infurze. W Rumble Fish mamy kilka projektów, w których jakiś komponent backendowy wchodzi w interakcje
z Ethereum blockchain. W niektórych przypadkach on reaguje na powiadomienia o eventach, w innych jest odpowiedzialny
za zamknięcie transakcji finansowej i musi działać szybko.

Nigdy nie jest dobrym pomysłem, aby proces o kluczowym znaczeniu dla firmy polegał na dobrej obsłudze zewnętrznej.
Gdy cokolwiek stanie się z Infurą, będzie to poza naszą kontrolą. Jeśli padnie, możemy tylko czekać. To może nie stanowić problemu dla wielu aplikacji, ale w naszym przypadku musimy wyeliminować takie ryzyko.

TL;DR podsumowanie
-------------

Aby postawić node'a, wykonaj poniższy punkt. Załóżmy, że masz skonfigurowane konto AWS
z programistycznym dostępem.


0. Sklonuj ten repozytorium.

   ::

     git clone https://github.com/rumblefishdev/cf-parity-mainnet.git

1. Otwórz terminal. Zainstaluj ``aws`` and ``jq`` jeśli nie jest zainstalowane.


2. Ustal, z którym kontem i regionem współpracujesz.

   ::

      export AWS_DEFAULT_REGION=eu-central-1
      # set to matching entry in ~/.aws/credentials
      export AWS_DEFAULT_PROFILE=...

3. Utwórz repozytorium, zbuduj obraz i wypchnij go.

   ::

      bash build_and_upload.sh

Ta komenda tworzy repozytorium ECR na twoim koncie i przepycha lekko spersonalizowany obraz klienta Parity.   


4. Określ parametry node'a.

   ::

      cd cloudformation
      cp stack-parameters.default.json stack-parameters.json
      $EDITOR stack-parameters.json


   Tu trzeba określić:
    - ``VpcId`` to run the chain
    - ``DNSName`` to register for your node (eg. ``mainnet.rumblefishdev.com``)


5. Utwórz CloudFormation stack.

   ::

      bash -x create-stack


6. Idź do konsoli CloudFormation i zaczekaj aż się skończy tworzenie stacku.


Uzyskaj wyeksportowane dane wyjściowe ``NameServer`` i umieść je jako wpis NS w konfiguracji DNS twojej domeny.
 

7. Zaczekaj około 3 dni na zakończenie synchronizacji i weryfikacji.

8. Idź do konsoli AWS EC2 do sekcji Volumes i zrób volume snapshot.

9. Gdy snapshot jest gotów, umieść go jako ``ChainSnapshodId`` parameters ``stack-parameters.json``
   i zaktualizuj stack.

   ::

      bash -x update-stack


Wyzwania związane z uruchamianiem mainnet node'a
----------------------------------

Trwałość danych Blockchain 
&&&&&&&&&&&&&&&&&&&&&&&&&&&

Największym wyzwaniem związanym z uzyskaniem wmainnet node'a jest jego synchronizacja.
Synchronizacja nowo podłączonego node'a od zera zajmuje 2-3 godziny, aby dostać się do bieżących bloków.
Jeszcze 2-3 dni potrzebne, aby zakończyć proces weryfikacji kryptograficznej wszystkich bloków.
Proces weryfikacji korzysta ze wszystkich dostępnych IOPS, co sprawia, że ​​node jest mało responsywny.
W tym czasie node może ulec opóźnieniu nawet o 30 bloków przez co robi się mało przydatny. Dopiero po zakończeniu procesu weryfikacji node robi się stabilny i możemy na nim polegać. Stack'i oferowane przez AWS nie rozwiązują tego problemu - każdy node
zaczyna się od nowa. To jest ok, o ile nie masz dużej ilości danych do zsynchronizowania
z innych node'ów.

Ponieważ uzyskanie nowego nod'a "kosztuje" 3 dni, konieczne jest utrzymanie danych między uruchomieniami node'a. Oczywiście istnieje więcej niż jeden sposób, aby to osiągnąć. Rozwiązanie, które proponujemy to przechowywać dane bloków na dysku EBS i zrobić jego snapshot kiedy blockchain zostanie w pełni zsynchronizowany. Jeśli node zostanie zakończony, a nowy przejmie jego funkcje, wystarczy zsynchronizować bloki od momentu snapshota, a nie całe 3 lata historii blockchainu.

Oczywiście to nie jest idealne i wolelibyśmy mieć trwałe dane bez konieczności ponownego zsynchronizowania wszystkich bloków, więc do tej pory nie znaleźliśmy idealnego rozwiązania.


Podejścia, które zlekceważyliśmy
###############################

Pojedyńczy trwały EBS
+++++++++++++++++++++

Jedną z rzeczy, którą próbowaliśmy, było utworzenie trwałego EBS volume poza kontekstem maszyny EC2 i podłączenie go do node'a podczas uruchamiania. Takie podejście ma swoje zalety. Kiedy maszyna się wyłącza, to nowa się odpala w miejscu gdzie poprzednia maszyna się skończyła. To jest świetna funkcja, ponieważ minimizuje opóźnienie ponownej synchronizacji.

Z drugiej strony takie podejście nie działa dobrze z rosnącą liczbą instancji w górę i w dół.
W scenariuszu, w którym chcielibyśmy mieć więcej node'ów do przełączenia awaryjnego lub zrównoważenia obciążenia, trzeba dodać dodatkową warstwę, aby zdecydować, który dysk EBS mamy użyć lub w miare możliwości stworzyć nowy.
Odrzuciliśmy to podejście jako zbyt skomplikowane.


Elastic File System (EFS)
+++++++++++++++++++++++++

Kolejną interesującą próbą rozwiązania rosnącego problemu było użycie EFS. W przeciwieństwie do EBS ten system może być
połączony z wieloma instancjami, które dzielą go za pomocą protokołu podobnego do NFS. Niestety widzieliśmy
że node'y z blockchainowymi danymi na EFS bardzo długo się synchronizują. Parity używa dużo
IOPS i EFS oferuje znacznie niższą wydajność niż EBS.



Dostęp do publicznej sieci dla warstwy synchronizacji
&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&

Aby się zsynchronizować, node musi być w stanie akceptować połączenia z innych node'ów.
Mówiąc wprost, wymagane jest aby jedna strona połączenia mogła akceptować
połączenia, więc technicznie moglibyśmy obyć się bez dostępu do publicznej sieci. Jednak jakbyśmy pomineli publiczny dostęp,
nasz node mógł by pracować tylko z node'ami oferującymi dostęp publiczny, co eliminuje duży fragment poola partnerów.

Aby zapewnić publiczny dostęp, skorzystaliśmy z następujących kroków.

1. Parity jest uruchamione w kontenerze dokowania. Port 30303 jest połączony przez taki cloudformation stack.

   ::

     Resources:
       TaskDefinition:
         Type: AWS::ECS::TaskDefinition
         Properties:
           ...
           ContainerDefinitions:
             ...
             PortMappings:
               - ContainerPort: 30303
                 HostPort: 30303
                 Protocol: tcp


2. Node powinien znać swój publiczny adres IP, ponieważ jest używany jako identyfikator enode emitowany do
   innych node'ów. To rozwiązanie jest wyłącznie dla EC2 i opiera się na wewnętrznym API dostępnym z komputera. 

   From ``docker/run_parity.sh``:

   ::

      PUBLIC_IP=`curl -s http://169.254.169.254/latest/meta-data/public-ipv4`
      /parity/parity --config config.toml --nat extip:$PUBLIC_IP

3. Aby port maszyny EC2 był dostępny, należy go również otworzyć w konfiguracji grupy zabezpieczeń.
   Ta część stacku jest odpowiedzialna właśnie za to. 


   ::

     Resources:
       ECSSecurityGroup:
         Type: AWS::EC2::SecurityGroup
         Properties:
           ...
           SecurityGroupIngress:
             - FromPort: 30303
               ToPort: 30303
               CidrIp: 0.0.0.0/0
               IpProtocol: tcp



Prywatny dostęp do json-rpc i interfejsów websocket
&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&

Parity ma jeszcze dwa interfejsy sieciowe do uzyskiwania dostępu do danych blockchain.
  - port 8545 jest używany dla json-rpc api: umieszczenie transakcji i uzyskiwanie wszelkiego rodzaju informacji
  - port 8546 może być używany do otrzymywania powiadomień z node'a o nowych blokach i / lub eventach


Najpierw omówmy, dlaczego uważamy, że json-rpc nie powinien być publicznie dostępny. W zależności od konkretnego
przypadku otworzenie json-rpc może nie sprawiać problemu. Jednak w Rumble Fish wierzymy że cokolwiek
co może być ukryte powinno pozostać ukryte.

Pozostawienie otwartego interfejsu json-rpc nie stanowi zagrożenia dla pieniędzy. Przynajmniej póki nie ma podstawowego błędu w Parity, który nie został zidentyfikowany. Niemniej jednak łatwo sobie wyobrazić, że osoba atakująca może po prostu uruchomić wiele zapytań na node'zie, aby zapobiec jego prawidłowemu użyciu. Więc warto się postarać i zrobić tą część bezpieczniejszą.

Nasze podejście do prywatnego dostępu składa się z następujących elementów.

1. Cloudformation stack tworzy i eksportuje specjalną grupę SecurityGroup używaną do uzyskiwania dostępu do node'a.
   Możesz zaimportować inny stack używając: 

   ::

     !Fn::Import MainnetParity-AccessSecurityGroup

2. Ta grupa ma dostęp do instancji używając następującego ustawienia w grupie SecurityGroup
   Instancji EC2. 

   ::

     Resources:
       ECSSecurityGroup:
         Type: AWS::EC2::SecurityGroup
         Properties:
           ...
           SecurityGroupIngress:
             - FromPort: 8545
               ToPort: 8545
               SourceSecurityGroupId: !GetAtt AccessSecurityGroup.GroupId
               IpProtocol: tcp
             - FromPort: 8546
               ToPort: 8546
               SourceSecurityGroupId: !GetAtt AccessSecurityGroup.GroupId
               IpProtocol: tcp



Te porty te są kierowane do docker kontenera, podobnie do tego co wcześniej robiliśmy z portem 30303.    

    ::

      Resources:
        TaskDefinition:
          Type: AWS::ECS::TaskDefinition
          Properties:
            ...
            ContainerDefinitions:
              ...
              PortMappings:
                - ContainerPort: 8545
                  HostPort: 8545
                  Protocol: tcp
                - ContainerPort: 8546
                  HostPort: 8546
                  Protocol: tcp

3. Klient łączący się z json-rpc / websocketem musi używać prywatnego adresu IP instancji.
   Osiągamy to, tworząc Route53 HostedZone i rejestrując IP instancji tam przy odpalaniu. 

Cloudformation stack eksportuje serwery DNS tej strefy jako

   ::

     !Fn::Import MainnetParity-NameServer



 lub wyszukiwanie w eksporcie konsoli AWS.
  

Powinieneś umieścić tą wartość jako wpis NS w konfiguracji swojej domeny DNS.


Monitotrowanie i logowanie 
----------------------

Stack jest skonfigurowany do zbierania interesujących plików z maszyny i przesyłania ich do CloudWatch
log stream'u ``MainnetParity-logs``.


  .. image:: ./docs/images/cloudwatch-parity-logs.png
      :width: 80%
      :align: center



Proces synchronizacji i weryfikacji
&&&&&&&&&&&&&&&&&&&&&&&&&&&&&

Tutaj interesującymi bitami są nazwy plików ``/parity/parity/...`` które są wynikami procesu Parity. 
Przy pierwszym uruchomieniu stack użyje synchronizacji warp, aby pobrać historię blockchainu
przy użyciu protokołu pobierania zbiorczego Parity.

Na wyjściu to wygląda tak:

::

  2018-05-11T09:27:56.202Z ++ curl -s http://169.254.169.254/latest/meta-data/public-ipv4
  2018-05-11T09:27:56.253Z + PUBLIC_IP=18.196.95.41
  2018-05-11T09:27:56.253Z + /parity/parity --config config.toml --nat extip:18.196.95.41
  2018-05-11T09:27:56.297Z Loading config file from config.toml
  2018-05-11T09:27:56.350Z 2018-05-11 09:27:56 UTC Starting Parity/v1.10.3-stable-b9ceda3-20180507/x86_64-linux-gnu/rustc1.25.0
  2018-05-11T09:27:56.350Z 2018-05-11 09:27:56 UTC Keys path /root/.local/share/io.parity.ethereum/keys/Foundation
  2018-05-11T09:27:56.350Z 2018-05-11 09:27:56 UTC DB path /root/.local/share/io.parity.ethereum/chains/ethereum/db/906a34e69aec8c0d
  2018-05-11T09:27:56.350Z 2018-05-11 09:27:56 UTC Path to dapps /root/.local/share/io.parity.ethereum/dapps
  2018-05-11T09:27:56.350Z 2018-05-11 09:27:56 UTC State DB configuration: fast
  2018-05-11T09:27:56.350Z 2018-05-11 09:27:56 UTC Operating mode: active
  2018-05-11T09:27:56.361Z 2018-05-11 09:27:56 UTC Configured for Foundation using Ethash engine
  2018-05-11T09:27:56.730Z 2018-05-11 09:27:56 UTC Public node URL: enode://ec52f4ae94c624b1f8bf9c9b60fd63261beb42af6fea9d0fa4aeb6f52047fdf4afd92d9e3cd9c0f3387e892f378b3491ed8d85c38349ad50dce99539e952e38f@18.196.95.41:30303
  2018-05-11T09:27:57.057Z 2018-05-11 09:27:57 UTC Updated conversion rate to Ξ1 = US$694.89 (6852745.5 wei/gas)
  2018-05-11T09:28:06.806Z 2018-05-11 09:28:06 UTC Syncing       #0 d4e5…8fa3     0 blk/s    0 tx/s   0 Mgas/s      0+    0 Qed        #0    1/25 peers      8 KiB chain    3 MiB db  0 bytes queue   10 KiB sync  RPC:  0 conn,  0 req/s,   0 µs
  2018-05-11T09:28:16.806Z 2018-05-11 09:28:16 UTC Syncing snapshot 9/1370        #0    2/25 peers      8 KiB chain    3 MiB db  0 bytes queue   10 KiB sync  RPC:  0 conn,  0 req/s,   0 µs
  2018-05-11T09:28:21.807Z 2018-05-11 09:28:21 UTC Syncing snapshot 15/1370        #0    2/25 peers      8 KiB chain    3 MiB db  0 bytes queue   10 KiB sync  RPC:  0 conn,  0 req/s,   0 µs
  2018-05-11T09:28:26.808Z 2018-05-11 09:28:26 UTC Syncing snapshot 21/1370        #0    2/25 peers      8 KiB chain    3 MiB db  0 bytes queue   10 KiB sync  RPC:  0 conn,  0 req/s,   0 µs
  2018-05-11T09:28:31.809Z 2018-05-11 09:28:31 UTC Syncing snapshot 27/1370        #0    3/25 peers      8 KiB chain    3 MiB db  0 bytes queue   10 KiB sync  RPC:  0 conn,  0 req/s,   0 µs
  2018-05-11T09:28:36.809Z 2018-05-11 09:28:36 UTC Syncing snapshot 29/1370        #0    3/25 peers      8 KiB chain    3 MiB db  0 bytes queue   10 KiB sync  RPC:  0 conn,  0 req/s,   0 µs



Proces synchronizacji snapshotów zajmuje około 3 godzin. Po zsynchronizowaniu snapshotów Parity pobierze wszystkie bloki utworzone od ostatniego snapshotu, aż do obecnie najnowszego bloku.
Ta faza wygląda tak:

::

  2018-05-11T10:26:46.793Z 2018-05-11 10:26:46 UTC Syncing snapshot 1327/1370        #0   26/50 peers      8 KiB chain    3 MiB db  0 bytes queue   10 KiB sync  RPC:  0 conn,  0 req/s,   0 µs
  2018-05-11T10:26:56.798Z 2018-05-11 10:26:56 UTC Syncing snapshot 1346/1370        #0   26/50 peers      8 KiB chain    3 MiB db  0 bytes queue   10 KiB sync  RPC:  0 conn,  0 req/s,   0 µs
  2018-05-11T10:27:08.097Z 2018-05-11 10:27:08 UTC Syncing #5590000 b084…309c     0 blk/s    0 tx/s   0 Mgas/s      0+    0 Qed  #5590000   24/25 peers     63 KiB chain    1 KiB db  0 bytes queue    6 MiB sync  RPC:  0 conn,  0 req/s,   0 µs
  2018-05-11T10:27:16.794Z 2018-05-11 10:27:16 UTC Syncing #5590000 b084…309c     0 blk/s    0 tx/s   0 Mgas/s   1750+    1 Qed  #5591752   26/50 peers    174 KiB chain   39 KiB db   95 MiB queue   11 MiB sync  RPC:  0 conn,  0 req/s,   0 µs

Wykonanie tego etapu zajmie jeszcze około godzinę.

Po zakończeniu tej fazy log zmieni się w następujący sposób:

::

  2018-05-11T15:24:30.011Z 2018-05-11 15:24:30 UTC Syncing #5595608 f2fe…d003     0 blk/s    0 tx/s   0 Mgas/s      0+    7 Qed  #5595619   11/25 peers     33 MiB chain  182 MiB db    1 MiB queue    8 MiB sync  RPC:  0 conn,  0 req/s,   0 µs
  2018-05-11T15:24:41.386Z 2018-05-11 15:24:41 UTC Updated conversion rate to Ξ1 = US$679.41 (7008882.5 wei/gas)
  2018-05-11T15:24:41.795Z 2018-05-11 15:24:41 UTC Imported #5595620 ef95…d8b2 (181 txs, 7.98 Mgas, 4237.27 ms, 27.63 KiB) + another 3 block(s) containing 330 tx(s)
  2018-05-11T15:24:48.290Z 2018-05-11 15:24:48 UTC Imported #5595622 221b…509d (162 txs, 7.99 Mgas, 1194.76 ms, 25.13 KiB)
  2018-05-11T15:24:51.186Z 2018-05-11 15:24:51 UTC Imported #5595623 b744…cf9c (183 txs, 7.98 Mgas, 1698.02 ms, 33.23 KiB)
  2018-05-11T15:25:27.225Z 2018-05-11 15:25:27 UTC     #40653   13/25 peers     37 MiB chain  182 MiB db  0 bytes queue   24 MiB sync  RPC:  0 conn,  0 req/s,   0 µs
  2018-05-11T15:25:27.241Z 2018-05-11 15:25:27 UTC     #40653   13/25 peers     37 MiB chain  182 MiB db  0 bytes queue   24 MiB sync  RPC:  0 conn,  0 req/s,   0 µs
  2018-05-11T15:25:27.252Z 2018-05-11 15:25:27 UTC     #40653   13/25 peers     37 MiB chain  182 MiB db  0 bytes queue   24 MiB sync  RPC:  0 conn,  0 req/s,   0 µs
  2018-05-11T15:25:27.310Z 2018-05-11 15:25:27 UTC     #40653   13/25 peers     37 MiB chain  182 MiB db  0 bytes queue   24 MiB sync  RPC:  0 conn,  0 req/s,   0 µs
  2018-05-11T15:25:41.464Z 2018-05-11 15:25:41 UTC Imported #5595627 a4a9…9dc0 (136 txs, 7.98 Mgas, 529.92 ms, 19.68 KiB)
  2018-05-11T15:26:02.263Z 2018-05-11 15:26:02 UTC     #78637   23/25 peers     37 MiB chain  183 MiB db  241 KiB queue   22 MiB sync  RPC:  0 conn,  0 req/s,   0 µs
  2018-05-11T15:26:03.398Z 2018-05-11 15:26:03 UTC Reorg to #5595628 8fc3…7c58 (a4a9…9dc0 18c7…4d47 #5595625 f6c1…feae 3faf…012d af04…83a8)

Nowy typ linii logowania rozpoczynający się od numeru bloku (``#40653 ..``) pochodzi z procesu weryfikacji pobranych bloków. W tym procesie Parity weryfikuje każdy blok kryptograficzny i zapewnia, że ​​nikt nie manipuluje danymi.

Ten proces trwa około 3 dni, gdy jest uruchamiany na``t2.machine`` with gp2 EBS ``300 IOPS``. 
Podczas jego działania można obserwować w monitorowaniu EBS że wszystkie dostępne IOPS są zużywane. Zrzut ekranu poniżej przedstawia moment zakończenia procesu weryfikacji. Możesz zobaczyć różnicę we wzorcu użycia.

.. figure:: docs/images/read-iops-end-of-sync.png
    :width: 80%

    Read IOPS

.. figure:: docs/images/write-iops-end-of-sync.png
    :width: 80%

    Write IOPS


Ponieważ proces weryfikacji jest ograniczony IO, można go przyspieszyć, wyposażając dysk EBS w dodatkowe IOPS.
W naszym CloudFormation stack
używamy ``gp2`` VolumeType with the size of ``100 GB``. AWS zapewnia 300 podstawowych IOPS dla takiego dysku. 
Jeśli chcesz przyspieszyć weryfikację, możesz zmodyfikować VolumeType na ``io1`` and give it ``1200`` IOPS. 
Na tym poziomie obserwujemy, że proces weryfikacji nie jest już ograniczony przez dostępne IOPS, ale brakuje mu mocy CPU.
Dlatego możesz przepchnąc go na inny poziom, zmieniając rozmiar maszyny EC2 z ``t2.medium`` to ``c5.large``.
Działając na ``c5.large`` zauważyliśmy, że podczas weryfikacji Parity używa 2000 IOPS i może zakończyć cały proces w około 7 godzin, więc jest to dobry skrót, jeśli chcesz szybko uzyskać wyniki. 
Pamiętaj, że skonfigurowane IOPS nie są tanie - miesięczny koszt pozostawienia dysku o tym rozmiarze oraz IOPS, będzie w zasięgu 100 USD, więc bądź ostrożny.

Pomysł jest taki, że po zakończeniu synchronizacji i weryfikacji można zrobić snapshota i użyć go do ponownego uruchomienia klastra za pomocą
zmniejszonego dysku i typu maszyny.


Pozostańie zsynchronizowanym
&&&&&&&&&&&&&&&

Gdy node jest w pełni zsynchronizowany, zwykle pozostaje zsynchronizowany z najnowszym blockiem :-)

.. image:: docs/images/parity-diff-to-infura.png
    :width: 80%


Powyższy obrazek przedstawia efekt wywołania ``eth_blockNumber`` na naszym node'ie i na Infurze.
Przez większość czasu node'y są zsynchronizowane. Sporadycznie nasz node lub Infura spada o 1-4 bloki do tylu.

Pamiętaj, że obecnie ten repozytorium nie zawiera Lambdy odpowiedzialnej za gromadzenie
powyższych danych. Zostanie to uwzględnione w przyszłych artykułach.

.. [1] https://docs.aws.amazon.com/blockchain-templates/latest/developerguide/blockchain-templates-ethereum.html
