# Implementierung von Google OAuth 2.0

## Einleitung

Im Kern ist LetterFlow ein Rich-Content-Editor. Es bietet einem **Autor** eine Reihe von **Erweiterungen**, um ein **Dokument** zu erstellen oder zu bearbeiten.

Ein **Dokument** ist ein Container, der strukturierte Daten über einen **Beitrag** speichert. Dokumente werden in einer **Cloud Firestore-Sammlung** innerhalb eines verbundenen **Firebase-Projekts** gespeichert.

Der Unterschied zu herkömmlichen Text-Editoren (üblicherweise WYSIWYG - "what you see is what you get") ist der, dass die Repräsentation der Daten (Text, Bilder, Videos, Code-Snippets usw.) rein im JSON-Format (de)serialisiert wird. Das gibt Endnutzern die volle Kontrolle über die Darstellung ihrer Inhalte (Posts), während die Daten eben dieser in einem einheitlichen Datenmodell repräsentiert werden. Schließlich können Nutzer mit dem Client ihrer Wahl ihre Beiträge abrufen und ihre individuellen Designs anpassen.

## Technologischer Stack

Der jetzige Stack für die LetterFlow-Anwendung ist wie folgt:

- Rust mit Tauri
- Angular (inkl. Tailwind CSS)
- Firebase mit Google Cloud Platform Integration (GCP)

Zur Verbindung der Benutzer mit ihren Daten wird Google OAuth 2.0 verwendet, da es nahtlos mit GCP integriert ist.

## Problemstellung

Auf dem Papier klingt die Aufgabe ziemlich einfach:

Ein User lädt die LetterFlow-Desktop-App herunter, meldet sich "Mit Google anmelden" an und kann dann über das Interface "rich-content" Inhalte erstellen.

Es wird sich jedoch herausstellen, wie umfangreich eine Integration von OAuth 2.0 in einer mit Tauri entwickelten Desktop-Anwendung werden kann.

## Schnittstelle und Datenmodelle

Ein zentraler Punkt der Desktop-Applikation ist, neben der Struktur der Datenmodelle ("Autor", "Beitrag" etc.), die Schnittstelle zwischen der Anwendung und den Nutzern sowie ihren Daten zu schaffen. Ohne eine generische Lösung für die Verbindungsherstellung zwischen Nutzern und deren Cloud Firestore-Daten ist die Nutzbarkeit der Anwendung hinfällig.

## Alternativen zur Autorisierung

Für eine generische, skalierbare Lösung in Sachen Autorisierung bzw. Authentifizierung gibt es neben OAuth auch andere Spezifikationen, über deren Einsatz ich während des Grundaufbaus der Anwendung nachgedacht habe. Die meisten (und zumindest fürs Erste) habe ich jedoch als "overkill" angesehen. Die folgende Liste beschreibt eine Reihe von Spezifikationen und warum diese letzten Endes nicht bevorzugt wurden bzw. die Implementierung obsolet ist:

### API-Schlüssel

API-Schlüssel sind eine einfache Methode zur Authentifizierung, bei der eine eindeutige Zeichenfolge verwendet wird, um den Zugriff auf eine API zu ermöglichen. Diese Methode ist leicht zu implementieren, bietet jedoch weniger Sicherheit als die oben genannten Standards, da sie oft nicht die gleiche Granularität oder den gleichen Schutz bietet.

Diese Methode kam für mich von vornherein nicht in Frage, da ich dafür eine eigene Datenbank managen müsste. Dadurch würde ich die Flexibilität der Anwendung in Sachen Nutzung und Sicherheit schmälern. Hinzu kommt: Sollte der API-Schlüssel aus irgendeinem Grund verloren gehen oder in einem Data-Breach landen, habe ich ganz schnell andere, unberechenbare Probleme zu lösen.

### OpenID Connect (OIDC)

OpenID Connect ist eine Schicht über OAuth 2.0, die eine einfache Möglichkeit bietet, Benutzer zu authentifizieren und grundlegende Profilinformationen bereitzustellen. Es ermöglicht, dass Anwendungen Identitätsinformationen von einem Identitätsanbieter (Identity Provider, IdP) abrufen, ohne dass der Benutzer seine Anmeldedaten mehrfach eingeben muss. OIDC verwendet JWT (JSON Web Tokens) für den Austausch von Informationen und bietet eine Standardisierung der Authentifizierungsprozesse.

Für die LetterFlow-Anwendung ist OpenID obsolet, da es in Google OAuth 2.0 integriert ist. Außerdem ist OIDC alleine nicht ausreichend für die Operationen, die die Desktop-Anwendung durchführen würde. Profilinformationen sind zwar Teil des "Auth-Flows" und der Benutzeroberfläche, jedoch eine unzureichende Berechtigung für die Kommunikation mit der Google Cloud Platform.

### FIDO (Fast Identity Online)

FIDO ist eine Initiative zur Entwicklung sicherer Authentifizierungsprotokolle, die Passwörter ersetzen sollen. FIDO verwendet öffentliche Schlüssel und ermöglicht eine starke, benutzerfreundliche Authentifizierung, die auf biometrischen Daten (wie Fingerabdruck oder Gesichtserkennung) basieren kann. Die FIDO-Protokolle bieten eine Alternative zu herkömmlichen Methoden der Benutzeranmeldung und -verifizierung.

Ich selbst habe noch nie FIDO per Spezifikation integriert, sondern nur Methoden davon in ganz anderen Kontexten und Zwecken, wie z.B. GPG. In meinen Augen ist diese Form der Authentifizierung sehr zukunftsträchtig, weil immer mehr Geräte mit der nötigen Hardware auf dem Markt kommen, die die Massentauglichkeit einer solchen Methode ermöglichen. Da ich Sorge hatte, dass manche Nutzer meiner Desktop-App nicht die neuesten Endgeräte besitzen und auch wegen der komplexeren Implementierung von FIDO, habe ich beschlossen, diese Methode in die "nice-to-have"-Feature-Liste einzutragen.

### SAML (Security Assertion Markup Language)

SAML ist ein XML-basiertes Standardprotokoll, das hauptsächlich für Single Sign-On (SSO) in Unternehmensanwendungen verwendet wird. Es ermöglicht den sicheren Austausch von Authentifizierungs- und Autorisierungsdaten zwischen einem Identitätsanbieter und einem Dienstanbieter. SAML ist besonders in großen Organisationen und für föderierte Identitätssysteme beliebt.

Da LetterFlow einen Firebase- bzw. Google Cloud Platform-Account erwartet, ist es eher nicht sinnvoll, SAML zu implementieren. Zum einen bin ich keine (große) Organisationseinheit, zum anderen verwalte ich keine eigene Infrastruktur. Ich kann mir aber vorstellen, dass im Falle einer Weiterentwicklung der Applikation SAML eine interessante und womöglich passendere Alternative zu OAuth wäre. Jedoch müsste die Weiterentwicklung des zugrunde liegenden Systems der Desktop-Anwendung so überarbeitet werden, dass mehrere Cloud-Anbieter (speziell Cloud-Datenbank-Anbieter) adaptiert werden können. Eine weitere Entwicklungslinie, in der SAML von Vorteil wäre, ergäbe sich, wenn die Anwendung über eine eigens verwaltete Server-Infrastruktur funktionierte. Dies ist gerade nicht der Fall, und die jetzigen Features kommen sehr gut mit OAuth aus.

### JWT (JSON Web Tokens)

Während JWT häufig in Kombination mit OAuth verwendet wird, kann es auch als eigenständige Methode zur Authentifizierung und Autorisierung dienen. JWT ermöglicht es, Informationen zwischen Parteien sicher zu übertragen, indem es JSON-Daten in einem kompakten Format kodiert, das digital signiert werden kann. JWTs sind besonders nützlich in modernen Webanwendungen, da sie eine zustandslose Authentifizierung ermöglichen.

Basierend auf dem System-Design meiner Anwendung ist JWT (auch) eine passende Lösung. Es war von vornherein mein Anspruch, Nutzern mehr Kontrolle über die Granularität der erteilten Berechtigungen für GCP-Services geben zu wollen. Zugleich wollte ich mir Schwierigkeiten ersparen, wenn es in Zukunft darum ginge, Zugriff auf andere GCP-Ressourcen zu ermöglichen bzw. einzufordern. Ein Beispiel hierfür wäre die Verbindung mit der Google Cloud Scheduler Resource, die es ermöglicht, Cron-Jobs zu erstellen. Diese Resource käme bei einem Feature zum Einsatz, das Nutzern erlaubt, planmäßig (an einem bestimmten Datum) Inhalte zu veröffentlichen. Dieselbe Funktionalität wäre natürlich auch mit JWTs allein möglich - für Microservices ist dieser Ansatz womöglich der idealste.

In einem idealen Szenario würde die Desktop-Anwendung sowohl OAuth als auch JWT verwenden. Für die initiale Authentifizierung und Autorisierung (von GCP-Ressourcen) käme OAuth 2.0 zum Einsatz. Danach, wenn dieser Flow erfolgreich war, würden alle HTTP-Anfragen einen JWT im "Authorization"-Header inkludieren (typischerweise als "Bearer"-Token).

Glücklicherweise ist genau das der Fall mit Googles OAuth 2.0 Implementierung.

## Ablauf des OAuth 2.0 Flows

Was passiert nun während eines OAuth 2.0 Flows innerhalb der Desktop-Anwendung?

Wenn ein User die Anwendung startet, wird im Hintergrund überprüft, ob eine Authentifizierungs-Sitzung (Session) aus einer vorherigen Nutzung der Anwendung existiert. Session-Daten werden mit Hilfe von **keyring** (ein Rust Crate) lokal gespeichert. **keyring** nutzt die plattform-spezifische Implementierung, um Anmeldeinformationen zu speichern - auf Windows-Plattformen nutzt es den "Windows Credential Manager", auf Linux den DBus-basierten "Secret Service" oder "kernel keyutils" und auf macOS den "local keychain". Bei erstmaligem Nutzen und willkürlich alternierend muss der User diesem Verfahren (nochmal) explizit zustimmen.

Die Datenstruktur für eine Session inkludiert einen OAuth 2.0 Token (*access token*) und gegebenenfalls einen *refresh token*. Darüber hinaus werden grundlegende Informationen zum Benutzer gespeichert, wie zum Beispiel der Name und das zuletzt ausgewählte Firebase-Projekt.

Existiert eine vorherige Session und enthält diese einen noch gültigen *access token*, wird der OAuth 2.0 Flow übersprungen und der User erhält Zugang zu seiner/n Dokument/en.

### Zugriffstoken und Erneuerung

Um einen *access token* zu erhalten, muss der User explizit zustimmen, dass die LetterFlow-Anwendung auf seine GCP-Ressourcen zugreifen darf. Daraufhin wird eine HTTP-Anfrage an den Google OAuth 2.0 Authorisierungs-Endpunkt gesendet, um die Zustimmung zu erhalten. Falls der Benutzer zustimmt, wird ein *authorization code* zurückgegeben.

Mit dem *authorization code* muss nun ein *access token* angefordert werden. Dies geschieht in einer zweiten HTTP-Anfrage an den Token-Endpunkt von Google, bei der die Client-ID, der *authorization code*, der *redirect uri* und das Client-Secret (das im Backend gespeichert werden muss) übergeben werden. Im Erfolgsfall gibt Google ein *access token* und eventuell ein *refresh token* zurück.

LetterFlow speichert und verwendet diese Tokens, um auf die Cloud Firestore-Ressourcen des Benutzers zuzugreifen.

Das *access token* hat jedoch eine begrenzte Lebensdauer (in der Regel 1 Stunde), daher ist es wichtig, es rechtzeitig zu erneuern. Dies geschieht mit dem *refresh token*, der, sofern er in der ersten Anfrage zurückgegeben wurde, dazu verwendet werden kann, ein neues *access token* zu generieren, ohne dass der Benutzer sich erneut anmelden muss.

Falls das *refresh token* auch abgelaufen ist oder nicht mehr gültig ist, muss der Benutzer den OAuth 2.0 Flow erneut durchlaufen.

### Authentifizierung und Autorisierung in LetterFlow

Die Anwendung geht davon aus, dass Nutzer sich mit einem Google-Konto anmelden, dem Firebase-Ressourcen zugeordnet sind. Da Nutzer mehrere Google-Konten besitzen können und diese gegebenenfalls in der Liste der auswählbaren Konten im OAuth-Flow auftauchen können, macht es natürlich keinen Sinn, sich mit einem Konto anzumelden, dem diese Zuordnung fehlt.

Authentifizierung und Autorisierung sind komplexe Prozesse, vor allem, wenn es nicht nur auf *einer* Plattform funktionieren muss. Im Falle von LetterFlow hat es einige Wochen an Testing, Feinjustierung und Anpassung gebraucht. Es sind immer noch einige Punkte offen, die ich noch einmal überdenken bzw. überwachen muss, damit ich mit weniger Sorgen den ersten stabilen Release in die Wege leiten kann.

Dieser Prozess allein hat grob geschätzt 30-40 Prozent der Zeit für den Aufbau der Anwendung eingenommen. Es sind noch einige UI-relevante Anpassungen vonnöten, damit der OAuth-Flow für die Endnutzer innerhalb der Anwendung wirklich "im Flow" ist. Damit meine ich konkret den Weiterleitungsprozess von und zu den Google OAuth-Servern.

Die OAuth 2.0-Spezifikation gibt vor, dass Nutzer zu einer *success* URL weitergeleitet werden, wenn der Anmeldeprozess erfolgreich beendet wurde. Es gibt unterschiedliche OAuth-Flow-Arten, aber die Mechanismen für den Anmeldeprozess sind bei allen Flow-Arten die selben.

Zum Beispiel ist es möglich, sich mit Hilfe von Pop-Ups anzumelden. Dabei öffnet sich ein Browser-Fenster, das zu dem jeweiligen OAuth-Anbieter weiterleitet. Nutzer wählen dann das gewünschte Anmelde-Konto aus und werden nach Erteilung der Berechtigung für die jeweilige Applikation zu dieser weitergeleitet.

Damit Pop-Up OAuth-Flows funktionieren können, müssen native Anwendungen in der Lage sein, sogenannte "Deep-Links" zu handlen, ein Prozess, bei dem ein OAuth-Anbieter so konfiguriert ist, dass er bei der Weiterleitung nach einer Anwendung auf dem System des Nutzers sucht, die die Weiterleitung "abfängt". Für gewöhnlich können nur native Anwendungen Deep-Links implementieren, aber aus unterschiedlichen Gründen implementiert nicht jede Anwendung Deep-Links.

In LetterFlow kommt ein anderer Flow zum Einsatz. Ich nenne ihn den "in-app" Flow. Dieser erzeugt, wie ich finde, eine deutliche Verbesserung in Sache Benutzerfreundlichkeit.

Anstatt den OAuth-Flow in einem Pop-Up-Fenster durchzuführen, starten wir speziell für diesen Prozess einen temporären lokalen Server, der die Weiterleitungs-URL hostet. Dafür wird vor dem Start des OAuth-Flows eine zufällige, freie Port-Nummer generiert, um potenzielle Konflikte mit anderen Anwendungen zu vermeiden.

Bei der Initialisierung des OAuth-Clients in Rust konfigurieren wir die temporäre Weiterleitungs-URL und starten dann den Anmeldeprozess. Dadurch bleibt der User für den gesamten Anmeldeprozess auf der Anwendung. Sobald der OAuth-Flow erfolgreich beendet wurde, zerstören wir den temporären Server wieder.

## Fazit

Insgesamt ermöglicht die Implementierung von Google OAuth 2.0 in der LetterFlow-Anwendung eine sichere und benutzerfreundliche Möglichkeit zur Authentifizierung und Autorisierung. Durch die Verwendung von OAuth 2.0 und der Integration von JWTs kann die Anwendung sicher auf die Cloud Firestore-Ressourcen der Benutzer zugreifen und gleichzeitig eine nahtlose Benutzererfahrung bieten.

Die hier skizzierte Architektur und der Authentifizierungsprozess sind nicht nur eine Lösung für die aktuellen Anforderungen der Anwendung, sondern auch eine flexible Grundlage für zukünftige Erweiterungen und Verbesserungen.
