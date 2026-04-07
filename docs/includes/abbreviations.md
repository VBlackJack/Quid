<!-- Registry hives -->
*[HKLM]: HKEY_LOCAL_MACHINE — ruche systeme contenant la configuration globale de la machine
*[HKCU]: HKEY_CURRENT_USER — ruche contenant la configuration de l'utilisateur connecte
*[HKU]: HKEY_USERS — ruche contenant toutes les ruches utilisateur chargees
*[HKCR]: HKEY_CLASSES_ROOT — ruche fusionnee des associations de fichiers et objets COM
*[HKCC]: HKEY_CURRENT_CONFIG — ruche contenant le profil materiel actif

<!-- Registry value types -->
*[REG_SZ]: String — chaine de caracteres Unicode
*[REG_DWORD]: Double Word — entier 32 bits (0 a 4 294 967 295)
*[REG_QWORD]: Quad Word — entier 64 bits
*[REG_BINARY]: Binary — donnees binaires brutes
*[REG_EXPAND_SZ]: Expandable String — chaine avec variables d'environnement (%SystemRoot%, etc.)
*[REG_MULTI_SZ]: Multi-String — liste de chaines separees par des null

<!-- Group Policy & Active Directory -->
*[GPO]: Group Policy Object — objet de strategie de groupe
*[AD]: Active Directory — annuaire d'entreprise Microsoft
*[OU]: Organizational Unit — unite d'organisation dans Active Directory
*[ADMX]: Administrative Template XML — modele de strategie de groupe (definitions)
*[ADML]: Administrative Template Language — fichier de langue pour les modeles ADMX
*[RSOP]: Resultant Set of Policy — jeu de strategies resultant applique a une machine ou un utilisateur
*[LGPO]: Local Group Policy Object — outil Microsoft pour appliquer des GPO locales

<!-- Security -->
*[ACL]: Access Control List — liste de controle d'acces
*[SID]: Security Identifier — identifiant de securite unique pour un utilisateur, groupe ou machine
*[DPAPI]: Data Protection API — API de protection des donnees utilisateur (chiffrement)
*[LAPS]: Local Administrator Password Solution — rotation automatique du mot de passe administrateur local
*[UAC]: User Account Control — controle de compte utilisateur
*[MFA]: Multi-Factor Authentication — authentification multifacteur
*[SDDL]: Security Descriptor Definition Language — langage de definition des descripteurs de securite

<!-- Device management -->
*[MDM]: Mobile Device Management — gestion des appareils mobiles
*[CSP]: Configuration Service Provider — fournisseur de configuration pour MDM (Intune, etc.)
*[OMA-URI]: Open Mobile Alliance Uniform Resource Identifier — chemin de configuration MDM personnalise
*[SCCM]: System Center Configuration Manager — outil de deploiement et gestion Microsoft (ancien nom)
*[MECM]: Microsoft Endpoint Configuration Manager — nom actuel de SCCM
*[WSUS]: Windows Server Update Services — service de distribution de mises a jour Windows

<!-- Windows technologies -->
*[WMI]: Windows Management Instrumentation — infrastructure de gestion et supervision Windows
*[CIM]: Common Information Model — modele standardise de gestion (successeur de WMI)
*[COM]: Component Object Model — modele d'objets composants Microsoft
*[DCOM]: Distributed Component Object Model — COM etendu au reseau
*[ETW]: Event Tracing for Windows — systeme de tracage d'evenements haute performance
*[BCD]: Boot Configuration Data — donnees de configuration du demarrage Windows
*[MSI]: Microsoft Installer — format de package d'installation Windows
*[API]: Application Programming Interface — interface de programmation
*[DLL]: Dynamic Link Library — bibliotheque de liens dynamiques
*[WinRM]: Windows Remote Management — service de gestion a distance Windows (protocole WS-Management)

<!-- Infrastructure & roles -->
*[RDS]: Remote Desktop Services — services de bureau a distance
*[DFS]: Distributed File System — systeme de fichiers distribue
*[NPS]: Network Policy Server — serveur de strategies reseau (RADIUS)
*[VDI]: Virtual Desktop Infrastructure — infrastructure de postes de travail virtualises
*[VPN]: Virtual Private Network — reseau prive virtuel
*[IIS]: Internet Information Services — serveur web Microsoft
*[NFS]: Network File System — protocole de partage de fichiers reseau
*[DNS]: Domain Name System — systeme de resolution de noms de domaine
*[DHCP]: Dynamic Host Configuration Protocol — protocole d'attribution automatique d'adresses IP

<!-- Group Policy internals -->
*[GPMC]: Group Policy Management Console — console de gestion des strategies de groupe
*[AGPM]: Advanced Group Policy Management — gestion avancee des GPO avec versioning et approbation
*[CSE]: Client-Side Extension — extension cote client qui applique un type de strategie
*[LSDOU]: Local, Site, Domain, OU — ordre d'application des strategies de groupe
*[GPP]: Group Policy Preferences — preferences de strategie de groupe (non-tatouantes)
*[ILT]: Item-Level Targeting — ciblage par element dans les preferences GPO
*[MLGPO]: Multiple Local Group Policy Objects — strategies de groupe locales multiples
*[SYSVOL]: System Volume — partage reseau AD contenant les scripts et fichiers de strategie
*[WDAC]: Windows Defender Application Control — controle d'applications Windows Defender
*[PKI]: Public Key Infrastructure — infrastructure a cles publiques
*[FSLogix]: FSLogix — solution Microsoft de gestion de profils et conteneurs d'applications
*[UPD]: User Profile Disk — disque de profil utilisateur pour les sessions RDS
*[RSAT]: Remote Server Administration Tools — outils d'administration de serveur a distance
*[WUfB]: Windows Update for Business — service de mise a jour Windows pour les entreprises

<!-- Network & security protocols -->
*[SMB]: Server Message Block — protocole de partage de fichiers et d'imprimantes Windows
*[LDAP]: Lightweight Directory Access Protocol — protocole d'acces aux annuaires
*[TLS]: Transport Layer Security — protocole de chiffrement des communications reseau
*[SSL]: Secure Sockets Layer — predecesseur de TLS (obsolete, remplace par TLS)
*[NTLM]: NT LAN Manager — protocole d'authentification Windows legacy

<!-- Defender & endpoint security -->
*[ASR]: Attack Surface Reduction — regles de reduction de la surface d'attaque Microsoft Defender
*[HVCI]: Hypervisor-protected Code Integrity — integrite du code protegee par l'hyperviseur
*[VBS]: Virtualization-Based Security — securite basee sur la virtualisation (Credential Guard, etc.)
*[EDR]: Endpoint Detection and Response — detection et reponse sur les terminaux

<!-- Hardware & firmware -->
*[TPM]: Trusted Platform Module — puce de securite materielle pour le chiffrement et l'attestation
*[UEFI]: Unified Extensible Firmware Interface — interface firmware moderne remplacant le BIOS
*[BIOS]: Basic Input/Output System — firmware legacy d'initialisation du materiel

<!-- Active Directory advanced -->
*[ADCS]: Active Directory Certificate Services — autorite de certification integree a Active Directory
*[ADFS]: Active Directory Federation Services — service de federation d'identite
*[DC]: Domain Controller — controleur de domaine Active Directory

<!-- Monitoring & compliance -->
*[SIEM]: Security Information and Event Management — centralisation et correlation des evenements de securite
*[SOC]: Security Operations Center — centre operationnel de securite

<!-- Network -->
*[RADIUS]: Remote Authentication Dial-In User Service — protocole d'authentification reseau centralise
*[UNC]: Universal Naming Convention — convention de nommage reseau (\\serveur\partage)
*[WQL]: WMI Query Language — langage de requete pour WMI
*[LLMNR]: Link-Local Multicast Name Resolution — resolution de noms multicast locale (risque de securite)
*[NBT-NS]: NetBIOS over TCP/IP Name Service — service de noms NetBIOS (risque de securite)
*[DoH]: DNS over HTTPS — resolution DNS chiffree via HTTPS
*[WFP]: Windows Filtering Platform — plateforme de filtrage reseau Windows

<!-- Authentication -->
*[WHfB]: Windows Hello for Business — authentification biometrique/PIN d'entreprise Microsoft

<!-- Security standards & frameworks -->
*[CIS]: Center for Internet Security — organisation publiant des benchmarks de securite
*[STIG]: Security Technical Implementation Guide — guide de securisation du DoD americain
*[ANSSI]: Agence Nationale de la Securite des Systemes d'Information — autorite francaise de cybersecurite
*[SCT]: Security Compliance Toolkit — boite a outils Microsoft pour appliquer les baselines de securite

<!-- Group Policy internals -->
*[GPC]: Group Policy Container — objet Active Directory contenant les metadonnees d'une GPO

<!-- Miscellaneous -->
*[RBAC]: Role-Based Access Control — controle d'acces base sur les roles
*[CAL]: Client Access License — licence d'acces client Microsoft
*[NAP]: Network Access Protection — protection d'acces reseau (obsolete, remplace par 802.1X/NAC)
