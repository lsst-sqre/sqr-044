:tocdepth: 1

.. sectnum::

Abstract
========

The identity management component of the Science Platform holds the list of authorized users, their group information, bindings from those users to external authentication providers, and associated metadata for both users and groups, such as quotas and other limits.
This document sets out the requirements for that component.
It also flags the minimal requirements and the requirements already met by the current identity.lsst.org system.

Intended use
============

The requirements laid out here are intended to guide selection of software to implement the identity management component, including the decision to develop software locally or use an existing product or service.
The requirements describe the system as a whole, and need not all be met by the same software package or service.
The final architecture is expected to be some hybrid of third-party software, third-party services, and components developed locally by Rubin Observatory.

Each requirement is given an ``IDM`` identifier with a unique number for use in other documents.
These will not be reused when requirements are added or removed.
Some numbers may be missing if requirements are deleted.

`LDM-554 <https://ldm-554.lsst.io/>`__ defines the general requirements for the Science Platform.
An identity management system that meets the requirements of this document will help satisfy requirements DMS-LSP-REQ-0020 (Authenticated User Access) and DMS-LSP-REQ-0007 (Abide by the Data Access Policies) of that document.
Specific requirements for the identity management system that derive from other Science Platform requirements in LDM-554 are marked with the requirement identifiers from that document.

`LSE-279 <https://docushare.lsst.org/docushare/dsweb/ServicesLib/LSE-279/History>`__ defines the requirements for identity management for Rubin Observatory at a higher level.
The requirements here are consistent with the requirements in that document.

This is not a design or architecture document for the identity management component.
That will come in a later document once an implementation strategy has been chosen to satisfy these requirements.

The minimal requirements for a functioning system are flagged with **Core requirement**.
The requirements that are currently supported by identity.lsst.org are flagged with *Supported by identity.lsst.org*.
Requirements that are currently supported by `Gafaelfawr`_ in combination with identity.lsst.org are flagged with *Supported by Gafaelfawr*.
Both markers may have additional commentary if they do not apply to the entire requirement.

.. _Gafaelfawr: https://gafaelfawr.lsst.io/

Authentication
==============

Web authentication
------------------

IDM-0001
    Primary authentication to the Science Platform will be done via federated authentication.
    The Science Platform will not store a local password, second-factor code, or other primary authentication credential for any user.
    It will issue tokens used for API access, but those tokens will be issuable and revocable by each user after authenticating with the primary federated authentication provider.
    [DMS-LSP-REQ-0023]

    **Core requirement.**
    *Supported by identity.lsst.org and Gafaelfawr.*

IDM-0002
    For most prospective users, initial authentication in order to create a Science Platform account will be via the InCommon federation.
    Metadata provided by the prospective user's local identity provider via the InCommon federation may be used to determine affiliation in order to decide if a user should be able to create an account.

IDM-0003
    Prospective users may authenticate through another federated identity provider.
    In this case, their pending account request will be held for approval by someone with authority to determine their data and access rights.
    This flow will be used to handle collaborators who would not otherwise have the necessary affiliation to create an account.

    **Core requirement.**
    *Supported by identity.lsst.org* using a hack involving groups.
    Accounts that aren't in any groups cannot access the Science Platform and therefore act like pending account requests.
    However, identity.lsst.org doesn't support finding users without groups, and the user has to request which groups they need to be a member of.
    If this requirement is met, manual approval effort can compensate for not having IDM-0002.

IDM-0004
    Once a user has created an account, they may attach additional federated identities to the same account.
    This should require authentication via one of their current identity providers and then via the new identity provider for each new federated identity.
    Once this has been done, all such identities will map to the same account in the Science Platform.
    This allows users with multiple institutional affiliations to use whichever authentication provider is most convenient for them.
    [DMS-LSP-REQ-0024]

    *Supported by identity.lsst.org.*

IDM-0005
    Authenticated users must be able to view all federated identities attached to their account and revoke any of them.

    *Supported by identity.lsst.org.*

IDM-0006
    GitHub and Google must be supported as additional identity providers.
    They may be added as additional identities to an existing account via IDM-0004, or be used by collaborators as described in IDM-0003.

    *Supported by identity.lsst.org*, except that they cannot be used as the only mechanism for authentication.
    An NCSA account is still required for Science Platform access.

IDM-0007
    Users with administrative access to the Science Platform (for example, the ability to impersonate other users) must authenticate with two authentication factors.
    These will generally be a password or some equivalent, and some additional security token bound to a sandboxed device (not a general-purpose laptop), such as an OTP authenticator on a mobile phone.
    Since all authentication is via federated identity (IDM-0001), this must be implemented by requiring authentication from an external provider that, in turn, requires two-factor authentication.
    GitHub or Google (IDM-0005), with appropriate configuration, would be suitable.

IDM-0008
    The authentication session generated by a web authentication must follow reasonable practices for secure cookie storage.
    This means, at least, setting ``Secure`` and ``HttpOnly`` and using either opaque session tokens or cryptographically-signed data.

    It is unclear whether identity.lsst.org supports this requirement.
    Many cookies it sets are not ``HttpOnly`` and one is not flagged as ``Secure``.

IDM-0009
    If a user primarily uses non-affiliation-providing identity providers such as Google and GitHub, there must be a mechanism to force the user to reauthenticate using an affiliation-providing identity provider to reconfirm their affiliation.
    This shouldn't be done routinely, but may be necessary periodically to confirm that a user still has an appropriate affiliation.
    One possible mechanism to implement this may be to add an expiration date for the user that is automatically extended when the user logs on with an appropriate identity provider.
    We do not yet know how affiliation and access rights will be determined or managed, so we may not need this facility, but it must be available in case it is necessary.

IDM-0010
    Users must select their own account name during initial account creation.
    This account name need not have anything to do with their federated identity.
    This will be used as the username for Science Platform services and must not contain a ``@`` character (or any other character that causes issues for Science Platform services).
    The account name must not match the name of any existing account.

    **Core requirement.**
    *Supported by identity.lsst.org* via the NCSA account creation process.

IDM-0011
    Account names may not be reused except via (rare) administrative intervention to remove accounts that were erroneously created.

IDM-0012
    Users must be able to change their account name.
    This change must propagate to all Science Platform services.
    Science Platform services should therefore use a unique identifier rather than the account name (such as the numeric UID provided as part of the account metadata) wherever possible, and if not possible, must explicitly allow for account renaming.
    Each component that uses the username rather than a UID must therefore have a plan for how to handle renaming and must be able to handle renaming events.
    This includes changes to home directory paths within the Notebook Aspect.
    The identity management system must have the capability of notifying those services when accounts are renamed.

IDM-0013
    When creating a new account, the user must be prompted for a preferred email address, possibly prepopulated with information from the identity provider.
    If a user with that email address already exists, the user must be prompted if they're sure they want to continue or if instead they want to use the existing account.
    This requirement will hopefully reduce the risk of duplicate accounts for the same person.
    (Also see IDM-1101.)

    Not directly supported by identity.lsst.org, although since identity.lsst.org requires NCSA account creation, it's unlikely to have too much of a duplicate account problem.

Token authentication
--------------------

IDM-0100
    Users may generate access tokens to use with API calls to the Science Platform.
    Access tokens must have an associated name chosen by the user.

    **Core requirement.**
    Gafaelfawr can create tokens, but the UI needs work, the tokens cannot be named, and the tokens expire.

IDM-0101
    User access tokens must not grant access to any administrative Science Platform function or permit changes to a user's account metadata or authentication information.
    Specifically, an access token cannot be used to attach a new federated identity to an account, revoke a federated identity from an account, or change a user's email address or group membership.

    **Core requirement.**
    *Supported by identity.lsst.org.*

IDM-0102
    Users may set an expiration time on user-generated access tokens.
    By default, user-generated access tokens do not expire, although their permissions are tied to the user's own permissions and thus they may become unusable if the account is frozen or deleted or its access permissions change.

IDM-0103
    Internal Science Platform components may also generate temporary access tokens to facilitate multi-layer services.
    Internal temporary access tokens must expire in a reasonable length of time, such as shortly after the expected maximum duration of the operation for which they were intended.

IDM-0104
    Tokens should be scoped to restrict their power.
    However, the number of scopes should not be so large as to be overwhelming.
    A user should be able to easily choose the necessary scope of a token for common token-based workflows.
    User-visible scopes should be limited to at most a few dozen, preferably fewer.
    The available scopes for tokens may vary by user and must be restricted to the list of scopes that user has access to based on their group membership.

IDM-0105
    Users must be able to see a list of all their current access tokens, including the names, creation dates, expiration times (if any), and associated scopes (but not including the value of the token).
    This must include internal temporary access tokens, although those should be visually separated from user-created access tokens.

    **Core requirement.**
    Gafaelfawr has a basic token list but is missing lots of details.

IDM-0106
    Tokens must not contain a frozen representation of group membership or permissions.
    Updates to the group membership of a user's account should also apply to all tokens issued for that user, provided that the scope of the token allows access.
    Services that need to know a user's group membership must present the token to the identity management system and ask what groups the corresponding user is in.
    The answer may change over the lifetime of the token, but may be cached; see IDM-3002 for more information.
    See `SQR-039 <https://sqr-039.lsst.io/>`__ for more discussion.

IDM-0107
    Accounts that are pending or frozen may not create tokens.
    Existing tokens for accounts that are pending or frozen must not be accepted as valid authentication.

    **Core requirement.**

Logging
-------

IDM-0200
    All initial authentications must be logged.
    The log must include the external IP address of the authenticating client, information about the identity provided by the identity provider, and the mapped Science Platform account (if any).

    **Core requirement.**
    *Supported by Gafaelfawr.*

IDM-0201
    All changes to the authentication metadata, such as changes to federated identity bindings, must be logged.

IDM-0202
    All token authentications from outside the Science Platform must be logged.

    **Core requirement.**
    *Supported by Gafaelfawr.*

IDM-0203
    Users must be able to see their recent web authentications, at least including timestamp and external authentication provider.
    Ideally this should include GeoIP information for the IP address, although getting accurate data inexpensively can be challenging so this isn't a firm requirement.

IDM-0204
    When displaying the list of federated identities associated with the account, the date and time at which that identity was last used to authenticate must be shown alongside.

IDM-0205
    When displaying the list of user-generated tokens, the date and time at which a user-generated token was last used must be shown alongside the token name.

IDM-0206
    Users must be notified via email of any change to their linked federated identities or any creation or revocation of a new user-generated token.

Account management
==================

Status
------

IDM-1000
    Accounts that are pending approval (under IDM-0003) can authenticate and see their account status and metadata page, but not access any other part of the Science Platform.
    They may not create tokens.

    **Core requirement.**
    *Supported by identity.lsst.org* via a group hack.

IDM-1001
    Administrators of the Science Platform must be able to freeze accounts.
    Frozen accounts may be placed in a state where they cannot authenticate at all, or in a state where they can only see their account status and metadata page but no other part of the Science Platform.
    A reason viewable by other administrators should be associated with a frozen account.
    The reason may contain any non-control UTF-8 character.
    Frozen accounts still hold the account name and do not allow it to be reused.

    **Core requirement.**
    *Supported by identity.lsst.org* mostly, using the hack of removing the user from groups, but it's awkward, doesn't revoke tokens, and still allows access to and changes to the account metadata.

IDM-1002
    Administrators of the Science Platform must be able to delete accounts.
    This is normally used for mistakenly-created accounts, not for accounts that were legitimate and active but should no longer be allowed access.

    This may be supported on identity.lsst.org, although not with the privileges the Science Platform administrators currently have.

IDM-1003
    It must be possible to set an expiration date on an account.
    This can be done by Science Platform administrators, or by the person approving access in the IDM-0003 use case.
    When the expiration date arrives, the account must be automatically frozen.

IDM-1004
    Users must be notified via email of upcoming account expiration so that they can investigate renewal options if needed.

Metadata
--------

IDM-1100
    A full name must be associated with each account and prepopulated with information from the identity provider.
    The user must be able to change the full name to anything they wish.
    The full name may include any non-control UTF-8 character.

    **Core requirement.**
    *Supported by identity.lsst.org* although it doesn't support prepopulation, and we have not tested UTF-8.

IDM-1101
    An email address must be associated with each account, chosen during account creation, and prepopulated with information from the identity provider if available.
    The user must be able to change the email address to anything they wish, but they must then verify that the email address is valid and owned by them by responding to a challenge sent to that email address.
    The old email address must also receive a notification of the change that allows the change to be canceled or reported as fraudulent.
    Challenges for an email address must not contain user-provided content so that they cannot be used for spamming purposes.

    **Core requirement.**
    *Supported by identity.lsst.org* in general, although some of the specifics of the confirmation flow may not be exactly as described.

IDM-1102
    Each account must be associated with information about how their eligibility was determined, including whether this was done via an automated process or by manual approval.
    This eligibility information must include the date that eligibility was last determined, and may include a date at which eligibility needs to be reviewed.

IDM-1103
    Each account must be tagged with one or more user class markers: US and Chile users with inherent data rights, users with data rights controlled by a Memorandum of Understanding, and Rubin Observatory project members.
    An account may be in more than one user class at the same time.
    This may be automatically populated during account creation.

IDM-1104
    An optional institutional affiliation may be affiliated with each account.
    This should be automatically populated from federation metadata on account creation.

    identity.lsst.org allows the user to record an affiliation, but doesn't automatically populate it.

Quotas
------

IDM-1200
    Users may have one or more quota grants associated directly with their account.
    These may represent file storage quotas or any other service limit that may vary by user (API rate limits, CPU equivalents for batch jobs, download size limits, or whatever may eventually be appropriate).
    The identity management system need not understand the quotas, but it should be able to sum multiple quotas under the same label.

IDM-1201
    The user must be able to view all of their existing quotas.

IDM-1202
    The user must be able to request a new quota grant.
    That request should be routed to some approval process by a manager of the relevant resource, who can then grant or deny the request via the identity management web interface.

IDM-1203
    Quota grants may expire.
    The user must be notified via email of pending quota grant expirations.

Administration
--------------

IDM-1300
    Administrators of the Science Platform must be able to modify any of the user's metadata on behalf of the user.

    This may be possible on identity.lsst.org.

IDM-1301
    Administrators of the Science Platform must be able to set and change expiration dates on accounts.

IDM-1302
    Administrators of the Science Platform must be able to approve a pending change of email address even if the user has not responded to the challenge.

IDM-1303
    Administrators of the Science Platform must be able to create, revoke, and change the expiration dates on quota grants.

IDM-1304
    Administrators must be able to impersonate a user and see the same thing that a user would see in the user metadata interface.

IDM-1305
    Administrators must be able to impersonate a user to other Science Platform services so that an administrator can debug issues that only affect a single user.

IDM-1307
    Administrators must be able to revoke user and internal temporary access tokens.

IDM-1308
    Administrators must be able to view a list of all accounts newly created within a given time period along with the mechanism by which their eligibility was determined.
    This may be used, for example, to perform subsequent manual review of accounts that were authorized via an automated process.

IDM-1309
    Administrators must be able to review the eligibility of accounts and update the determination and review dates.

IDM-1310
    Administrators must be able to change the user class and institutional affiliation of a user.

IDM-1311
    Administrators must be able to merge two accounts that are discovered retroactively to correspond to the same person.
    Such merges are expected to be rare and thus may be somewhat manual.
    For example, a merge may be done by copying group memberships from one account to another and then freezing the account that will no longer be merged.
    There must be some mechanism to mark an account explicitly as having been merged into another account.

Logging
-------

IDM-1400
    All changes to account metadata must be logged.
    If the changes were made by an administrator instead of the user, this must be clearly indicated in the logs.

IDM-1401
    All changes to quotas associated with users must be logged.

IDM-1402
    Users must be able to see a history of all of their quota changes.

IDM-1403
    All administrative user impersonation events must be logged, even if the administrator took no actions after impersonating the user.

IDM-1404
    All changes to user or internal access tokens (creation and revocation) must be logged, including associated metadata such as name, expiration, and scope.
    If the changes were made by an administrator instead of the user, this must be clearly indicated in the logs.

Groups
======

Management
----------

IDM-2000
    Users may be members of zero or more groups.

    **Core requirement.**
    *Supported by identity.lsst.org.*

IDM-2001
    Groups can be configured to control membership based on attributes provided by the identity provider.
    Membership in those groups must be tied to affiliation information from specific identity providers and dynamically adjusted if an authentication from that identity provider stops returning the same metadata.
    Multiple identity providers may provide access to the same group.
    In this case, the membership should only be withdrawn if all those identity providers stop providing the relevant information.
    It must be possible to periodically force users to authenticate with an attribute-providing identity provider to reconfirm access to their groups, similar to IDM-0009.

IDM-2002
    Users must be able to create their own groups.
    The owner of the group must then be able to add and remove members as they wish.
    Owners must also be able to add additional group owners who can then also control membership in the group.

    identity.lsst.org allows some people to create groups, but not all users.
    It is possible to add additional group owners.

IDM-2003
    It must be possible to create groups whose membership can only be changed by Science Platform administrators.

    **Core requirement.**
    *Supported by identity.lsst.org.*

IDM-2004
    Groups must be checked against namespace rules that, for instance, force all groups created by a user to start with a specific prefix that includes their username.
    Group names must be composed of only non-control ASCII characters (not UTF-8 since this may cause interoperability problems with other consumers of the group name).

IDM-2005
    Group membership may include an expiration date.
    When the expiration date is reached, the user will be automatically removed from the group.
    Anyone who can control membership in the group must be able to update the expiration date.

IDM-2006
    Owners must be able to rename groups while preserving all quota grants and membership.
    Groups must therefore be assigned a unique identifier (GID) that does not change when the group is renamed.
    Science Platform services should use that identifier rather than the group name wherever possible.
    If not possible, Science Platform services must be prepared for groups to be renamed and handle that appropriately, similar to the requirements for renaming users given in IDM-0012.

IDM-2007
    Owners must be able to delete groups.
    This will need to trigger special handling in the Science Platform to handle group-owned data, so should also support configuring a warning before deleting a group and publishing an event of some type when a group is deleted.

IDM-2008
    Users must be able to see all of their group memberships and their expirations (if any).

    identity.lsst.org shows group membership but doesn't support expiration.

IDM-2009
    It must be possible to set the owner of a group to be another group.

Quotas
------

IDM-2100
    Groups may have one or more quota grants associated with the group.
    These are of two types: Quotas for the group itself (such as for shared storage space), and quota that is inherited by every member of the group (granting additional personal quota).
    These may represent file storage quotas or any other service limit that may vary by user (API rate limits, CPU equivalents for batch jobs, download size limits, or whatever may eventually be appropriate).
    The identity management system need not understand the quotas, but it should be able to sum multiple quotas under the same label.

IDM-2101
    All members of the group must be able to view all of its quota grants.

IDM-2102
    All owners of the group must be able to request new quota grants.
    That request should be routed to some approval process by a manager of the relevant resource, who can then grant or deny the request via the identity management web interface.

IDM-2103
    Quota grants may expire.
    The owners of a group must be notified via email of pending quota grant expirations.

Administration
--------------

IDM-2200
    Administrators must be able to create groups, delete groups, rename groups, and change the membership of any group.

IDM-2201
    Administrators must be able to change the quota grants and requests for any group.

IDM-2202
    Administrators must be able to impersonate a user and see exactly the same group management and display screens that the user would see.

Logging
-------

IDM-2300
    All group creation, deletion, renaming, and membership changes must be logged.
    If the changes were made by an administrator instead of a group owner, this must be clearly indicated in the logs.

IDM-2301
    Owners must be able to see, via the web interface, all history of changes to the group.

IDM-2302
    All changes to quotas associated with groups must be logged.

IDM-2303
    Group owners must be able to see a history of all changes to their group quotas.

IDM-2304
    Users must be able to see a history of all changes to their group membership.

IDM-2305
    All administrative user impersonation events must be logged, even if the administrator took no actions after impersonating the user.
    (This is covered by IDM-1403, but reiterating here since it applies to the group management screens as well.)

API
===

IDM-3000
    The identity management system must provide a read-only API to other Science Platform components.
    That API, when given a user authenticator (a token or cookie), must return the user metadata, group memberships, and individual and group quota information.

    **Core requirement** except for the quota pieces.
    *Supported by Gafaelfawr.*

IDM-3001
    All actions possible for an administrator to perform in the identity management system must be available via an administrative API as well.
    This should use separate authentication credentials from user-issued tokens for administrative users.

IDM-3002
    Science Platform components may cache the results of read-only API calls to the identity management system, including such information as group membership for a given token and user.
    The validity of that cache sets a bound on how quickly a token can be revoked.
    Science Platform components should refresh that information every five minutes, and no less frequently than once per hour.
    They should not query for the same information from the same token more frequently than every thirty seconds.
    The identity management system must be able to handle this volume of queries.

IDM-3003
    The identity management system must provide an API for creating a new quota request.
    This may be used as part of more complex workflows such as the submission of a science proposal.

Scaling
=======

IDM-4000
    The identity management system must be able to handle 10,000 active users and 50,000 total users including disabled and frozen users.

    *Supported by identity.lsst.org.*

IDM-4001
    The identity management system must be able to handle 10,000 active groups and 10,000 members of a single group.

    Unknown whether identity.lsst.org supports this.

IDM-4002
    The identity management system must be able to retain history of group membership changes for twenty years at a rate of 10 changes per day (100,000 records).

    Unknown whether identity.lsst.org supports this.

IDM-4003
    The identity management system must allow a single user to be a member of 50 groups.

    Unknown whether identity.lsst.org supports this.
