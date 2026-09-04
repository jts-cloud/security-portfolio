# Investigating a confused Deputy Attack

## Scenario
Reconstructed a five-stage OAuth consent-phishing kill chain in a live Azure tenant through forensic analysis of two linked app registrations.

## Environment
Live multi-user Azure training tenant, Reader access

## Investigation
1) 

The core. Numbered steps IN YOUR OWN WORDS: what you looked at, what you found, what you concluded at each step. 6 to 12 screenshots of meaningful moments (portal views, query results, before/after).

## What broke / what surprised me
I was surprised that a standard user can register an app by default, and that owning an app registration is essentially considered an unlogged priviledge path that an audit of Global Admins would completely miss. 

## Findings and recommendations
An attacker had phished a user's session token and utilized the user's ownership access to a legacy app to establish a service principal as a Owner belonging to a second,attacker-created app. The attacker also created a new client secret that would last indefinitely. The attacker attempted to keep persistent access to the legacy app by creating a custom api for the legacy app to a newly created rouge app that would attempt to phish for a delegated user token for the exposed api via malicious redirect URI. If the attacker had successfully phished for a delagated token for the exposed API, they would be able to act as the user on the legacy apps backend code to run a sync that used the priviledged API permissions.

My recommendations would be to:
* Revoke the client secret
* Remove the rouge service principal from Owners
* Delete the custom expose API Scope
* Revoke the OAuth2PermissionGrant, because containment does not remove it
* Remove the Attacker Redirect URI
* Review and reduce the Graph application permissions
* Disable default user app registration
* Audit every app registration's Owners list
* Configure an alert on new client secrets and new redirect URIs

## What I learned
* Conducting audits on user permissions is highly important. This scenario could have potentially been entirely prevented if user permissions were routinely being checked. 
* Running default settings DOES NOT mean they are the most secure settings. It is very dangerous to allow by default users to register apps and own that app registration. This is a situation where it is generally better to have either the IT or cybersec team to handle app registration and provide users access to apps upon approved requests. 
* Rotating user credentials and client secrets isn't enough to consider the environment secured. A through investigation needs to occur to ensure all assests are secure and there isn't any exposed or malicous content still in the environment. 