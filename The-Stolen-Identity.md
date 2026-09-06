# Investigating an Identity-Based Breach

## Scenario
Reconstructed a five-stage OAuth consent-phishing kill chain in a live Azure tenant through forensic analysis of two linked app registrations.

## Environment
Live multi-user Azure training tenant, Reader access

## Investigation
1) ENTRY. A user was phished, completed MFA, and had the resulting session token stolen. Since the session token was already authenticated, the environment treated the attacker as an authenticated user, bypassing conditional access. The phished user was also an Owner on a legacy connector app.

2) ESCALATE. Using the Owner rights, the attacker created a new client secret on the legacy app. That secret let them authenticate through the client credentials flow as the service principal itself, inheiriting the app's directory permisisons without ever signing in as a user. 
![Image showing Client Secret is set to year 2099](https://github.com/jts-cloud/security-portfolio/blob/e87b2f8b2942efd350a51cdc59995bcd3d673209/Images/Lab02/Lab02-img1.png)

3) PIVOT. A secret expires when it gets rotated. The attacker had registered their own app (every standard user can do this by default in Entra) and added its service principal to the legacy app's Owners List. Now they can re-credential the legacy app forever, even after the first secret is caught. 

![Image showing Rouge App](https://github.com/jts-cloud/security-portfolio/blob/e87b2f8b2942efd350a51cdc59995bcd3d673209/Images/Lab02/Lab02-img2.png)

4) PERSIST. The attack had created a backup plan: a custom scope published on the legacy app's Expose an API blade. This turns the legacy app into a callable backend resource, which means the attacker's own app can request delgated access to it.  
![Image showing Rouge scope set by attacker](https://github.com/jts-cloud/security-portfolio/blob/e87b2f8b2942efd350a51cdc59995bcd3d673209/Images/Lab02/Lab02-img3.png)

5) LOOT. Finally, a redirect URI on the rouge app pointing at an attacker-controlled infrastructure. Combining the rouge app's client ID, that redirect URI, and the exposed API scope crafts a working phishing URL. A victim who is already signed in on a corperate device clicks Accept on a consent prompt, and the authorization code lands on the attacker's server. 

![Image showing working attacker's phishing URL](https://github.com/jts-cloud/security-portfolio/blob/e87b2f8b2942efd350a51cdc59995bcd3d673209/Images/Lab02/Lab02-img4.png)

Even though the attacker's methods seem a bit convoluted, there is a method to the madness. Ordinary credential phishing runs the risk of being thwarted by device compliance, MFA prompting for authentication, and location rules. Consent phishing bypasses all of it, because the victim is already authenticated on a trusted device. The resulting OAuth2PermissionGrant is not removed by a password reset, not removed by revoking sessions, and not removed by enforcing MFA. Most standard containment playbooks leave it in place. This type of attack is called a confused deputy attack, where a trusted tool (automation script,administrative tool, or a priviledged service account) that's manipulated into executing a malicous command outside of it's intended function.      

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
