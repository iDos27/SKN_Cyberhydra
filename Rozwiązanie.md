# 🎤 CTF Cyber.Mil Zadanie z wolumenem i sekretami

Część rozwiązania oznaczona jako "Co mówisz" została utworzona przez sztuczną inteligencje na podstawie komend, których użyłem do rozwiązania zadania. Są jedynie przykładem dla osób, które nie pamiętają z prezentacji

---

## 💥 CZĘŚĆ 1: Volume Hunter

**Co mówisz:**
> "Zaczynamy od pierwszego scenariusza. Udało nam się przejąć plik konfiguracyjny (kubeconfig) jednego z deweloperów. Nazywa się `config4`. Zobaczmy, z czym mamy do czynienia."

**Co robisz (Terminal):**
```bash
kubectl --kubeconfig=config4 auth can-i --list
```
**Co mówisz:**
> "Używam komendy auth can-i --list, żeby sprawdzić moje uprawnienia. Wygląda tego dużo, ale te linki na dole to tylko publiczne endpointy API. Moje jedyne prawdziwe uprawnienie to tworzenie i listowanie Podów (kontenerów). Spróbujmy w takim razie pobrać listę sekretów albo wolumenów, żeby poszukać flagi."

**Co robisz (Terminal):**
```bash
kubectl --kubeconfig=config4 get secrets
```
**Co mówisz:**
> "Dostajemy Forbidden. Jesteśmy zamknięci w klatce. Nie mamy praw do plików na klastrze. Ale zaraz... Skoro mogę tworzyć dowolne kontenery, to co, jeśli stworzę kontener, który będzie moim koniem trojańskim? Pokażę Wam pewien plik."

**Co robisz (Terminal):**
```bash
cat local-exploit.yaml
```

**Co mówisz:**
> "To jest manifest naszego złośliwego Poda. Tworzymy zwykłego linuxa (busybox), ale spójrzcie na dół, na sekcję volumes. Używamy tu opcji hostPath: /. To krytyczny błąd konfiguracji klastra. Kubernetes pozwala nam wziąć CAŁY główny dysk twardy serwera (węzła) i zamontować go wewnątrz naszego małego, niepozornego kontenera w katalogu /host. Zobaczmy to w praktyce."


**Co robisz (Terminal):**
```bash
kubectl --kubeconfig=config4 apply -f local-exploit.yaml
kubectl --kubeconfig=config4 get pods
```

**Co mówisz:**
> "Nasz exploit działa, status to Running. Teraz wbijamy się do środka naszego Poda, otwierając w nim powłokę shell."

**Co robisz (Terminal):**
```bash
kubectl --kubeconfig=config4 exec -it local-explorer -- sh
```

**Co mówisz:**
> "Znak zachęty się zmienił. Jesteśmy w kontenerze. Zobaczmy, co mamy w katalogu /host."

**Co robisz (Terminal):**
```bash
ls -la /host
```

**Co mówisz:**
> "Bingo. Widzimy tu pliki konfiguracyjne całego serwera Linux, na którym stoi Kubernetes. Mając taki dostęp, mógłbym przeczytać plik /host/etc/shadow z hasłami, albo poszukać tokenów innych aplikacji. W naszym CTF-ie twórca schował flagę w folderze /var/tmp/sekretny_folder/. Ponieważ jesteśmy w podmontowanym dysku, wchodzimy tam dodając prefix /host."

**Co robisz (Terminal):**
```bash
cat /host/var/tmp/sekretny_folder/flag.txt
```

**Co mówisz:**
> "Mamy pierwszą flagę! Przełamaliśmy izolację kontenera. W realnym świecie przed takim atakiem chronią narzędzia takie jak Pod Security Admission, które zablokowałyby możliwość użycia hostPath. Zamykamy ten kontener i lecimy do drugiego zadania."

**Co robisz (Terminal):**
```bash
exit
```

## 🕵️‍♂️ CZĘŚĆ 2: Poszukiwacz sekretów (Secret Hunter)

**Co mówisz:**
> "W drugim zadaniu wcielamy się w innego użytkownika. Zdobyliśmy plik config2. Sprawdzamy, na co pozwala nam RBAC tym razem."

**Co robisz (Terminal):**
```bash
kubectl --kubeconfig=config2 auth can-i --list
```

**Co mówisz:**
> "Sytuacja odwrotna. Nie mogę tworzyć Podów, nie mam dostępu do dysku. Mam za to pełen odczyt zasobu secrets. Zobaczmy, co tam leży."

**Co robisz (Terminal):**
```bash
kubectl --kubeconfig=config2 get secrets
```

**Co mówisz:**
> "Mamy tu trzy sekrety: flag, ok oraz newtls. Jako napaleni hakerzy od razu rzucamy się na ten o nazwie flag. Kubernetes domyślnie przechowuje sekrety w formacie Base64, więc musimy to zdekodować w locie."

**Co robisz (Terminal):**
```bash
kubectl --kubeconfig=config2 get secret flag -o jsonpath='{.data.flag}' | base64 -d
```

**Co mówisz:**
> "Otrzymaliśmy ciąg ZWNzY3tUUllfSEFSREVSfQ==. Dlaczego to nadal wygląda jak Base64? Ponieważ twórca zadania złośliwie zakodował tekst w Base64 jeszcze zanim wrzucił go do Kubernetesa! Kubernetes zakodował to po raz drugi. Musimy użyć podwójnego dekodowania."


**Co robisz (Terminal):**
```bash
kubectl --kubeconfig=config2 get secret flag -o jsonpath='{.data.flag}' | base64 -d | base64 -d
```

**Co mówisz:**
> "ecsc{TRY_HARDER}. Klasyczny trolling twórców CTF-ów. Podobnie jest z drugim sekretem."

**Co robisz (Terminal):**
```bash
kubectl --kubeconfig=config2 get secret ok -o jsonpath='{.data.flag}' | base64 -d | base64 -d
```

**Co mówisz:**
> "ecsc{this_is_not_this_flag}. Okej, nie tędy droga. Zwróćcie uwagę na ostatni sekret. Ma typ kubernetes.io/tls. To nie jest zwykły plik tekstowy, to certyfikat kryptograficzny. Wyciągnijmy go."

**Co robisz (Terminal):**
```bash
kubectl --kubeconfig=config2 get secret newtls -o jsonpath='{.data.tls\.crt}' | base64 -d
```

**Co mówisz:**
> "Dostaliśmy surowy blok certyfikatu. Wiele osób tutaj się poddaje, bo widzi tylko kryptograficzny bełkot. Ale my wiemy, że certyfikaty x.509 posiadają metadane – informacje o tym, kto go wydał i dla kogo. Przepuścimy ten certyfikat przez narzędzie openssl, żeby go rozpakować na czytelny tekst i poszukamy pola Subject."

**Co robisz (Terminal):**
```bash
kubectl --kubeconfig=config2 get secret newtls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout
```

**Co mówisz:**
> "I oto jest nasza flaga! Została sprytnie wstrzyknięta w atrybut Common Name (CN) certyfikatu TLS. Podsumowując: w Kubernetesie informacje mogą wyciekać z wielu miejsc. Raz będzie to źle zabezpieczony wolumen, raz metadane certyfikatu. Kluczem jest metodyczne podejście i rozumienie, jak system jest zbudowany pod spodem. Dziękuję za uwagę!"
