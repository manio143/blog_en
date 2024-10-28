---
title: 3. Implementing privacy compliance
images: []
---

# 3. Implementing privacy compliance

In the modern internet everyone and their mother is collecting data about users.
And to be fair, I also want to collect some for legitimate reasons - making a better product and a better experience to my users.
But because there's a lot of nefarious actors out there we now have regulations in place that explicitly require services to provide users with more control over their data.

[GDPR](https://www.cloudflare.com/learning/privacy/what-is-the-gdpr/) and [ePrivacy directive](https://www.cloudflare.com/learning/privacy/what-is-eprivacy-directive/) are the EU legislation which mandate consumer privacy rights.
Their existence has spawned such services as [Plausible](https://plausible.io/blog/legal-assessment-gdpr-eprivacy) and [Umami](https://umami.is/) which offer compliant web analytics.

This blog will try to cover how I'm looking to implement privacy measures in Excos to comply with GDPR.
Mind you, there's no one way to do things compliantly, so depending on the requirements of your system this approach may or may not be suitable.

## Legitimate interest & Consent

When it comes to collecting and processing personal data (which includes anything that could identify an individual) we either need to show that we have a legitimate interest, i.e. the collection is required for what the user is trying to accomplish using the service, or we need to gather consent from the user for the "extra" data use.

Excos is a platform providing remote application configuration, experimentation over application telemetry and decision automation.
Collecting user billing details for example would be a legitimate interest to enter into a contract with the user to provide them the services.

But often we are collecting personal data without the user explicitly providing us with it.
For instance, the country inferred from user's IP address for service telemetry can be considered a personal data point if it's stored next to a user identifier.
And often times we want to know how many unique users we deal with so having some sort of user identifier is necessary to calculate it.

Do we need to calculate unique users to provide the service? Generally, no.

So if we want to do processing of personal data that is not the required to provide a service to the user (that they want to be provided), we need to get their consent.
Let's look at the following requirements:

> * Consent must be freely given, specific, informed and unambiguous.
> * The data subject must also be informed about his or her right to withdraw consent anytime.
> The withdrawal must be as easy as giving consent.
> * The consent must be bound to one or several specified purposes which must then be sufficiently explained.
> * Consent cannot be implied and must always be given through an opt-in, a declaration or an active motion.
> * The controller is not allowed to switch from the legal basis consent to legitimate interest once the data subject withdraws his consent. This applies even if a valid legitimate interest existed initially. Therefore, consent should always be chosen as a last option for processing personal data.
> <br>
> [_source_](https://gdpr-info.eu/issues/consent/)

Let's say we're considering the following:

1. Collect personal data to provide the service:
    * We store the information on which user made a change to a configuration and we may display this information on the configuration page to other users in the same organization.
    * We store the user email address in order to be able to send them a notification they opt-into once the system detects a problem with the user's application.
2. Collect personal data to allow for debugging and customer support:
    * We store information on the user action in the telemetry so that in case of a system error we can trace back the user steps and reproduce them to understand and fix the issue.
3. Collect and use telemetry to understand product usage and improve the product:
    * We may collect some extra telemetry to paint a bigger picture of user behavior, which was not needed for debugging only.
4. Create a user profile for experience personalization:
    * We create a user profile that tells us about their product usage history which we use to alter the user experience via in-app suggestions.
5. Create a user profile for marketing purposes:
    * We create a user profile in order to target them with marketing messages, such as to upsell freemium users.
6. Create an audit log for security purposes:
    * We store the user details together with information on when they signed in or accessed a resource in the system. If we detect suspicious activity we may temporarily block the user account.

We want to process as much data under legitimate interest clause as we can.
From what I've read we can treat 3 of the above as such: 1. provide, 2. debug problems, 6. security.

Then we would want to collect consent for the other 3 purposes.
Each purpose is treated separately and must be opted-into by the user separately.
For example, using check-boxes on the sign up form and in the user settings (remember about removing consent).

I'm going to create a consent bit-map:

* 0 - no consent for non-required data
* 1 - consent for product improvement telemetry (`I`)
* 2 - consent for personalization profile (`P`)
* 4 - consent for marketing profile (`M`)

The bitmap value will be stored in the user settings and used when initializing the telemetry system, so that we can decide on whether to collect extra telemetry or not.
Moreover, the telemetry will be stamped with the consent information to create evidence of what the user consent was when the data was collected.
The user may later take back consent which means we can no longer process the previously collected data, but it doesn't have to be deleted (immutable telemetry).

This last bit is important - it means that any time we perform processing we need to know what is the latest consent state.
Since we're tagging all our logs with the consent bitmap, it should be possible to do `ARGMAX(time, consent_map)` to find out their current preference and filter the rest of the logs accordingly.

## Right to be forgotten

Under the GDPR the user has the right to request removal of their personal data.
They can request only specific data to be removed.

For this purpose we're going to implement the following:

1. User has a persistent account ID (`puid`)
    * This id is used for cases where despite data removal request the data will have to be retained for other legal reasons: e.g. security and billing (within some retention period).
    * This id is used for account access, authorization, tenant membership, etc.
    * Any identifiable information such as email, name, etc. is stored together with this ID.
2. User has an operational user ID (`oid`)
    * This id is used in events and databases to denote actions taken by the user.
    * This id is tied to a user alias that they agreed can be shared publicly.
    * Upon user closing their account, the connection between their persistent ID and operational ID is severed.
3. User has at least one telemetry ID (`tyid`)
    * This id is created by hashing the operational ID with a salt that the user can ask to be rotated, which de-links all previously collected for them telemetry. _Resulting in a UUIDv5._
    * This id is used in all telemetry instead of the operational ID, which preserves unique user semantics and facilitates debugging.
    * Upon user closing their account the salt is cleared or rotated. 

In order to prevent indirect identification, any other object IDs stored in a way where they can be linked to an operational ID also need to be hashed before being added to telemetry with a tenant level salt.

User's personal data, except where it needs to be kept for legal reasons should be erased.

## Right of access

The user has the right to request a copy of all data currently held on them by the system.
They should have an "easy" way to download their data and they can request it over email or other channels.

We already are going to have an OData based web API for programmatic access, but we also need a button in the user profile where they can request their data easily.
A database sourced export would cover anything available through the API, but in a simplified user-centric view.
For example we wouldn't export all the configuration fields the user has modified, but only the fact they they modified that specific config.

Then a second data export would mainly consist of exporting the telemetry.
But we don't need to export all telemetry, but only that pertains to what we know about the user actions.
If a user action results in multiple telemetry events describing the system's internal operations, we only surface to the user the action that they had taken.

To support this, all telemetry events must be annotated with the information on whether they are exportable or not.
All UX events directly initiated by the user are exportable.

A data pipeline needs to be put in place where once daily/weekly (depending on the export event volume) it takes all the pending requests and executes a query over the telemetry and the database that reads **all** data we can currently link to the user and puts them into a portable file format the user can open in a blob storage.
The user would be then emailed a secure link to access this file.
After 30 days the file would be deleted.

In order to prevent abuse, a user cannot issue a new request until the previous one is completed.

## Data classification

Dealing with personal data requires us to classify certain data fields to denote that what they contain is personal or can link to personal data.
Further more most debugging activities should not allow us (the host) to view all data the users are managing.
Since this is potentially a B2B service, we wouldn't want to for example see their config names which could leak upcoming features before they are announced.

As such the data must be classified.
We will leverage this [package](https://www.nuget.org/packages/Microsoft.Extensions.Compliance.Redaction/) to create data classification attributes and ensure any sensitive data is redacted before logging.

Classification:

* UII - User Identifiable Information
    * This covers any directly identifiable information such as email, name, address, IP address (in some cases)
* UPI - User Pseudonymous Identifier
    * This covers any identifier which could be indirectly used to reach UII such as user IDs, session IDs, object IDs (if those objects link to a single specific user ID)
* UDI - User Demographic Information
    * This covers any data about the users age group, country of residence, or other demographic elements, which would not identify a single individual if de-linked
* CC - Customer Content
    * Any content which the users upload to the system (e.g. config names) which may be sensitive
* OI - Organization Information
    * Non personal data related to the tenant (including tenant ID)
* SYS - System Metadata
    * Any data which is system specific and would not be considered personal nor can be used to link to personal data on its own (e.g. enums, certain object IDs)

No UII and CC can appear in the logs. Only processed UPIs (appropriately hashed) can appear in logs.
