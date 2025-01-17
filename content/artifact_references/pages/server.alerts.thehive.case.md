---
title: Server.Alerts.TheHive.Case
hidden: true
tags: [Server Event Artifact]
---

Create a TheHive case when monitored artifacts complete with results.  Add the ClientId, FlowId, and FQDN as tags to the case.  Add FQDN as an observable.
Much of this was borrowed from: https://gist.github.com/scudette/3a32abd19350c8fe3368661c4278869d

It is recommended to use the Server Metadata section to store credentials, instead of having to store directly inside the artifact.


```yaml
name: Server.Alerts.TheHive.Case
description: |
   Create a TheHive case when monitored artifacts complete with results.  Add the ClientId, FlowId, and FQDN as tags to the case.  Add FQDN as an observable.
   Much of this was borrowed from: https://gist.github.com/scudette/3a32abd19350c8fe3368661c4278869d

   It is recommended to use the Server Metadata section to store credentials, instead of having to store directly inside the artifact.

type: SERVER_EVENT

author: Wes Lambert - @therealwlambert

parameters:
  - name: TheHiveURL
    default: https://mythehive
  - name: VeloServerURL
    default: https://myvelo
  - name: ArtifactsToAlertOn
    default: .
    type: regex
  - name: DisableSSLVerify
    type: bool
    default: true

sources:
  - query: |
      LET thehive_key = if(
           condition=TheHiveKey,
           then=TheHiveKey,
           else=server_metadata().TheHiveKey)
      LET flow_info = SELECT timestamp(epoch=Timestamp) AS Timestamp,
             client_info(client_id=ClientId).os_info.fqdn AS FQDN,
             ClientId, FlowId, Flow.artifacts_with_results[0] AS FlowResults
      FROM watch_monitoring(artifact="System.Flow.Completion")
      WHERE Flow.artifacts_with_results =~ ArtifactsToAlertOn

      LET cases = SELECT * FROM foreach(row=flow_info,
       query={
          SELECT FQDN, parse_json(data=Content)._id AS CaseID FROM http_client(
          data=serialize(item=dict(
                title=format(format="Hit on %v for %v", args=[FlowResults, FQDN]), description=format(format="ClientId: %v\n\nFlowID: %v\n\nURL: %v//app/index.html?#/collected/%v/%v", args=[ClientId, FlowId, VeloServerURL, ClientId, FlowId,]), tags=[ClientId,FlowId, FQDN]), format="json"),
          headers=dict(`Content-Type`="application/json", `Authorization`=format(format="Bearer %v", args=[thehive_key])),
          disable_ssl_security=DisableSSLVerify,
          method="POST",
          url=format(format="%v/api/case", args=[TheHiveURL]))
       })

       SELECT * from foreach(row=cases,
       query={
          SELECT * FROM http_client(
          data=serialize(item=dict(data=FQDN, dataType="fqdn", message=FQDN)),
          headers=dict(`Content-Type`="application/json", `Authorization`=format(format="Bearer %v", args=[thehive_key])),
          disable_ssl_security=DisableSSLVerify,
          method="POST",
          url=format(format="%v/api/case/%v/artifact", args=[TheHiveURL, CaseID]))
       })

```
