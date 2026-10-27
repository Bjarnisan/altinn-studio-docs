---
headless: true
hidden: true
---

#### Utvid app-prosessen med et signeringssteg
Det må legges til et prosessteg i `App/config/process/process.bpmn`, som i eksemplet nedenfor.

{{% notice info %}}
Det anbefales å dra inn prosessteget via prosessdesigneren i Altinn Studio. Da får man generert BPMN-diagram som viser flyten i appen. Merk at prosessdesigneren forløpig ikke håndterer alle aspekter av konfigurasjonen, så det må suppleres med noe manuelt oppsett.
{{% /notice %}}

Signering består av to brukerhandlinger (user actions). Dersom Altinn-brukergrensesnittet brukes av appen, så vil disse bli kalt automatisk når man står i signeringssteget. Om kun API-et benyttes, så må disse kalles manuelt via `/actions`-endepunktet eller process next.
- `sign`: Selve signeringshandlingen.
- `reject`: Dersom det skal være mulig å avbryte signeringssteget, så må det også legges til en gateway for å styre hvor prosessen skal gå videre da.

```xml
<bpmn:task id="SigningTask" name="Signering">
  <bpmn:extensionElements>
    <altinn:taskExtension>
      <altinn:taskType>signing</altinn:taskType>
      <altinn:actions>
        <altinn:action>sign</altinn:action>
        <altinn:action>reject</altinn:action>
      </altinn:actions>
      <altinn:signatureConfig>
        <altinn:dataTypesToSign>
          <altinn:dataType>ref-data-as-pdf</altinn:dataType>
        </altinn:dataTypesToSign>
        <altinn:signatureDataType>signatures</altinn:signatureDataType>

        <!-- Vi har laget en standard validator som kan slås på her. -->
        <!-- Den validerer at påkrevd antall signaturer (minCount på signatur-datatypen) er oppfylt. -->
        <!-- Om denne ikke slås på, bør man skrive egen validering av signaturer. -->
        <altinn:runDefaultValidator>true</altinn:runDefaultValidator>

        <!-- Valgfritt: Hvis du ønsker å sende kvitteringer for utført signatur, -->
        <!-- kan du spesifisere din correspondence-ressurs i feltet under. -->
        <altinn:correspondenceResource>app-correspondence-resource</altinn:correspondenceResource>
      </altinn:signatureConfig>
    </altinn:taskExtension>
  </bpmn:extensionElements>
  <bpmn:incoming>SequenceFlow_1oot28q</bpmn:incoming>
  <bpmn:outgoing>SequenceFlow_1if1sh9</bpmn:outgoing>
</bpmn:task>
```

Hvis du har valgt å sende signeringskvitteringer ved å spesifisere en correspondence-ressurs, finner du flere detaljer om denne konfigurasjonen [her](/nb/altinn-studio/v8/guides/development/signing/runtime-delegated-signing/#konfigurere-milj%C3%B8-spesifikke-correspondence-ressurser)

#### Legg til en gateway etter signeringssteget
Når en bruker utfører `reject`, sletter appen signaturene uten å kontrollere signeringssteget. Deretter følger prosessen sekvensflyten ut av steget. Uten en gateway er det den samme flyten som steget bruker etter signeringen, så instansen går videre til neste steg uten signaturer. Prosessdesigneren i Altinn Studio legger `reject` på alle signeringssteg, men legger ikke til gatewayen, så den må du legge til selv.

Legg til en eksklusiv gateway etter signeringssteget. Den sender `reject` tilbake til steget der brukeren fyller ut skjemaet, og alt annet videre til neste steg. Legg også til den nye flyten som `<bpmn:incoming>` på steget du sender brukeren tilbake til.

{{< code-title >}}
App/config/process/process.bpmn
{{< /code-title >}}

```xml
<bpmn:sequenceFlow id="SequenceFlow_1if1sh9" sourceRef="SigningTask" targetRef="Gateway_Signing" />

<bpmn:exclusiveGateway id="Gateway_Signing">
  <bpmn:incoming>SequenceFlow_1if1sh9</bpmn:incoming>
  <bpmn:outgoing>Flow_Signing_Reject</bpmn:outgoing>
  <bpmn:outgoing>Flow_Signing_Next</bpmn:outgoing>
</bpmn:exclusiveGateway>

<!-- reject: tilbake til steget der brukeren fyller ut skjemaet -->
<bpmn:sequenceFlow id="Flow_Signing_Reject" sourceRef="Gateway_Signing" targetRef="Task_1">
  <bpmn:conditionExpression>["equals", ["gatewayAction"], "reject"]</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- Alt annet: videre til neste steg -->
<bpmn:sequenceFlow id="Flow_Signing_Next" sourceRef="Gateway_Signing" targetRef="EndEvent_1">
  <bpmn:conditionExpression>["notEquals", ["gatewayAction"], "reject"]</bpmn:conditionExpression>
</bpmn:sequenceFlow>
```

Bruk `notEquals` på flyten videre. Når brukeren sender inn etter signeringen, går prosessen videre uten noen handling. Med en betingelse som `["equals", ["gatewayAction"], "sign"]` passer da ingen flyt, og prosessen stopper med en feil.

#### Legg til datatyper for å lagre signeringsdata
Denne datatypen legges til i `dataTypes` i `App/config/applicationmetadata.json`.

Den benyttes av signeringssteget til å lagre de faktiske signaturene som genereres når brukeren utfører signeringshandlingen.

```json
{
  "id": "signatures",
  "allowedContentTypes": [
    "application/json"
  ],
  "allowedContributors": [
    "app:owned"
  ],
  "minCount": 1,
  "actionRequiredToRead": "signature-access",
  "actionRequiredToWrite": "signature-access"
}
```

Verdien av `id`-feltet _må_ samstemme med verdien som har blitt spesifisert i [signeringssteget](#utvid-app-prosessen-med-et-signeringssteg).

Det er viktig å sette `allowedContributors` til `"app:owned"`. Det gjør at disse dataene ikke kan redigeres via appens API, men kun av appen selv. Før versjon 8.6 var denne konfigurasjonen feilstavet `allowedContributers`.

Vi anbefaler å sette opp [databeskyttelse](/nb/altinn-studio/v8/guides/development/restricted-data/) for signaturobjektene, ved å spesifisere feltene for `actionRequiredToRead` og `actionRequiredToWrite` i definisjonen ovenfor. Hvis du ikke ønsker å sette opp dette, kan du fjerne disse feltene.

#### Tilgangsstyring
Gi `read`, `write` og eventuelt `sign` til den som fyller ut skjemaet. Andre som skal signere må også få `read` og `write`.

Mer informasjon om action attributter finner du [her](/nb/altinn-studio/v8/reference/configuration/authorization/#action-attributter).