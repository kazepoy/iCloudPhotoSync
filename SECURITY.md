# Audit de sécurité — iCloud Photo Sync 1.7.0

Audit réalisé le 2026-11-10, avant installation sur un NAS Synology.

- **Code source audité :** commit `6c3059f` (HEAD de `main`), identique au dépôt d'origine `Euphonique/iCloudPhotoSync`
- **Paquet audité :** `iCloudPhotoSync-1.7.0.spk` de la release GitHub 1.7.0 (tag `1.7.0`, commit `b48d970`)

**Conclusion : aucun code ni aucune dépendance malveillante n'a été trouvé.** Le paquet publié correspond au code source. Quelques faiblesses non malveillantes sont listées plus bas.

---

## 1. Provenance du dépôt

- Le dépôt cloné (`kazepoy/iCloudPhotoSync`) est au même commit (`6c3059f`) que `Euphonique/iCloudPhotoSync` (vérifié avec `git ls-remote`). Le contenu est donc identique.
- Tous les commits sont de l'auteur (Pascal Pagel / Euphonique). Les deux derniers ne modifient que `.github/FUNDING.yml`.

## 2. Code de l'application

Revue ciblée de `app/api.cgi`, `lib/handlers/*`, `lib/*.py`, `bin/*.py`, `service-setup.sh` et `pyicloud_ipd`. Les gros fichiers (`sync_engine.py`, `iCloudPhotoSync.js`) ont été passés au crible par mots-clés (appels réseau, exécution de commandes, accès fichiers, encodages suspects), pas relus ligne par ligne.

- **Connexions réseau :** uniquement vers Apple (`icloud.com`, `apple.com`, `icloud-content.com` et leurs équivalents `.cn`), vers `api.github.com` (vérification de version seulement, sans téléchargement ni installation automatique) et vers l'API DSM locale (`localhost:5000/5001`).
- **Droits :** le paquet tourne sous un utilisateur dédié sans privilèges (`conf/privilege`).
- **Accès à l'API :** chaque appel passe par `authenticate.cgi` (session DSM + SynoToken). Sans session, la réponse est 401.
- **Mot de passe Apple :** il n'est envoyé qu'à Apple, via le protocole SRP, où le mot de passe lui-même ne transite pas. Il n'est jamais stocké durablement : seulement pendant la double authentification, chiffré dans `/dev/shm` (mémoire vive), puis effacé. Les jetons de session sont stockés avec les droits `0600`.
- **Défenses en place :** limitation des tentatives de connexion, validation des identifiants de compte avant de les utiliser dans des chemins, liste fixe des fichiers de logs lisibles, liste blanche des domaines pour le proxy de miniatures et de téléchargement, échappement HTML dans l'interface, nettoyage des données personnelles dans l'export des logs.
- **Commandes système :** toujours appelées avec une liste d'arguments (`subprocess`), jamais via `shell=True`.

## 3. Bibliothèques Python embarquées (`lib/vendor/`)

Comparaison fichier par fichier avec les paquets officiels de PyPI :

| Bibliothèque | Version | Résultat |
|---|---|---|
| requests | 2.31.0 | Identique |
| urllib3 | 1.26.18 | Identique |
| idna | 3.6 | Identique |
| charset_normalizer | 3.3.2 | Identique |
| six | 1.16.0 | Identique |
| srp | 1.0.21 | Identique |
| certifi | 2024.02.02 | Identique, plus 2 certificats ajoutés (voir ci-dessous) |

Les deux certificats ajoutés à `certifi/cacert.pem` sont **Apple Root CA - G2** et **Apple Root CA - G3**. L'ajout est documenté dans le fichier (support d'iCloud Chine). Leurs empreintes SHA-256 correspondent à celles publiées par Apple :
- G3 : `63:34:3A:BF:B8:9A:6A:03:EB:B5:7E:9B:3F:5F:A7:BE:7C:4F:5C:75:6F:30:17:B3:A8:C4:88:C3:65:3E:91:79`
- G2 : `C2:B9:B0:42:DD:57:83:0E:7D:11:7D:AC:55:AC:8A:E1:94:07:D3:8E:41:D8:8F:32:15:BC:3A:89:04:44:A0:50`

`pyicloud_ipd` est une version modifiée du client iCloud d'`icloudpd`. Il a été relu manuellement : rien d'anormal.

## 4. Binaires précompilés (`bin/heif/{x86_64,aarch64,armv7}/`)

Les **36 binaires** (`heif-convert` et 11 bibliothèques, pour 3 architectures) sont **identiques bit à bit** (SHA-256) aux paquets officiels de Debian 12 « bookworm », téléchargés sur deb.debian.org et snapshot.debian.org :

| Fichier | Paquet Debian |
|---|---|
| heif-convert | libheif-examples 1.19.7-1~bpo12+1 (`heif-dec`) |
| libheif.so.1, plugin libheif-libde265.so | libheif1 / libheif-plugin-libde265 1.19.7-1~bpo12+1 |
| libde265.so.0 | libde265-0 1.0.11-1+deb12u2 |
| liblzma.so.5 | liblzma5 5.4.1-1 (pas les versions xz 5.6.0/5.6.1 piégées) |
| libtiff.so.6 | libtiff6 4.5.0-6+deb12u3 |
| libzstd.so.1 | libzstd1 1.5.4+dfsg2-5 |
| libjpeg.so.62 | libjpeg62-turbo 2.1.5-2 |
| libjbig.so.0 | libjbig0 2.1-6.1 |
| libLerc.so.4 | liblerc4 4.0.0+ds-2 |
| libdeflate.so.0 | libdeflate0 1.14-1 |

## 5. Vérification du `.spk` publié

- L'empreinte SHA-256 du fichier téléchargé correspond à celle affichée par GitHub : `0ea796f8a940944a688c5e40f534c1df1ab104ce96ccbb44bc9be61ac1c48d35`.
- Le paquet a été recompilé avec `build.sh` à partir du tag `1.7.0`, dans un conteneur `python:3.12-slim`. Empreinte obtenue : `7cd7e710…`. Il est normal qu'elle diffère : `tar`/`gzip` enregistrent les dates de création des fichiers, donc deux builds ne sont jamais identiques bit à bit. Les deux paquets ont donc été comparés **fichier par fichier** :

| Élément | Résultat |
|---|---|
| Scripts d'installation, `conf/resource`, icônes | Identiques |
| Code Python, bibliothèques, binaires | Identiques |
| `INFO` | Seule la somme de contrôle de `package.tgz` diffère (attendu) |
| `conf/privilege`, `app/index.html` | Seules les fins de ligne (CRLF/LF) diffèrent |
| `app/iCloudPhotoSync.js` | Le paquet officiel contient le commit suivant le tag (`5a25bc0`, interface de l'onglet Albums) |
| `lib/vendor/urllib3/packages/` | En plus dans le paquet officiel, identique à PyPI |
| `app/images/applephotos.png` | En plus dans le paquet officiel (image PNG, inoffensive) |
| 164 fichiers `.pyc` (caches Python oubliés au build) | Voir ci-dessous |

Les fichiers `.pyc` ont été vérifiés en recompilant les sources avec le même interpréteur (Python 3.10 et 3.14) puis en comparant les objets de code :
- **160 sur 164** sont identiques aux sources ;
- **4** (`pyicloud_ipd`, Python 3.14) sont d'anciens caches. Leurs en-têtes ne correspondent plus aux sources, donc Python ne les exécutera pas. Ils ne contiennent que des adresses Apple.

---

## Faiblesses relevées (non malveillantes)

1. **Fichier sudoers dans les scripts d'installation du `.spk`** (`build.sh`, `postinst`, `postupgrade`). Ces scripts tentent de créer `/etc/sudoers.d/iCloudPhotoSync` :
   ```
   iCloudPhotoSync ALL=(root) NOPASSWD: /usr/syno/sbin/synoshare
   iCloudPhotoSync ALL=(root) NOPASSWD: /usr/syno/bin/synopkg install /tmp/ics_update_*
   ```
   S'il s'appliquait, ce fichier permettrait à quiconque contrôle le compte du paquet de devenir root, via l'installation de n'importe quel paquet. **Aucun effet en pratique sur DSM 7.2+** (la version minimale exigée) : le paquet tourne en `run-as: package`, donc ses scripts n'ont pas le droit d'écrire dans `/etc`. La désinstallation supprime ce fichier. À retirer de `build.sh`.
2. **Dépendances Python anciennes, avec des CVE connues** :
   - requests 2.31.0 : CVE-2024-35195, CVE-2024-47081 ;
   - urllib3 1.26.18 : CVE-2024-37891, CVE-2025-50181 ;
   - idna 3.6 : CVE-2024-3651 ;
   - certifi 2024.02.02 : CVE-2024-39689.

   Impact faible dans ce contexte : pas de proxy, pas de `.netrc`, trafic uniquement vers Apple.
3. **Incohérence entre `requirements-pure.txt` et le dossier `vendor/`** : le premier demande `urllib3==2.2.1`, alors que le dépôt contient la 1.26.18. Comme `pip install --target` ne remplace pas les dossiers déjà présents, c'est la 1.26.18 qui finit dans le paquet.
4. **Binaires figés** : les bibliothèques Debian embarquées ne reçoivent pas les correctifs de sécurité (par exemple, libtiff est en deb12u3 alors que Debian est passé à deb12u4). Risque faible : elles ne traitent que les photos de l'utilisateur, venant d'iCloud.
5. **Fichiers en trop dans le build officiel** : les `.pyc` et `applephotos.png` viennent du dossier de travail de l'auteur, pas du dépôt.
6. **`INFO` indique « SynoCommunity » comme distributeur**, alors que le paquet n'est pas distribué par SynoCommunity.
7. **Points mineurs** :
   - Le proxy de miniatures ne valide que la première adresse et suit ensuite les redirections (il n'est accessible qu'avec une session DSM).
   - Un appel vers l'API DSM locale désactive la vérification TLS (connexion `localhost` uniquement).
