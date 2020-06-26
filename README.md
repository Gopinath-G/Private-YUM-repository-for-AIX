# Private-YUM-repository-for-AIX
In recent days AIX is supporting many open source projects like ansible, git, automake etc, and this requires different rpm package which comes with its own dependencies.
YUM is one of the well know rpm package manager in linux world which manages the rpm package installations by resolving dependencies internally.
In most of the corporate networks AIX servers are not exposed to internet, In such scenarios an alternate solution is required to configure repository for AIX
This Document describes procedure to configure private yum repository for AIX in internal network.


# Pre-requisite

•	RHEL/CentOS/AIX server with internet facing network

•	Check and install RPM and YUM Incase of AIX  (While in RHEL and CentOS YUM is default RPM package manger)

	  Check rpm.rte is installed, if not download and install. (http://ftp.software.ibm.com/aix/freeSoftware/aixtoolbox/INSTALLP/ppc/rpm.rte)
	  Check YUM is installed if not download and install. (https://public.dhe.ibm.com/aix/freeSoftware/aixtoolbox/ezinstall/ppc/yum_bundle.tar)
    
•	Httpd services

•	File system or directory with 15G free space (will mention as <repo_dir> in this document)

# Procedure

Follow Below steps to configure private YUM repository for AIX

•	Create AIX Repository

•	Synchronize and Create Local Repository

•	Publish local repository to AIX clients through HTTP

•	Configure AIX Client to point private YUM repository


# 1.	Create AIX repository
a.	Verify YUM Is installed in the server

        #rpm -qa | grep -i yum
    
b.	Create repo file under /etc/yum.repos.d in case of RHEL and CentOS in case of AIX add below repositories /opt/freeware/etc/yum/yum.conf

        #vi /etc/yum.repos.d/aix.repo
    
c.	Update the IBM AIX repositories in aix.repo file

        [AIX_Toolbox]
        name=AIX generic repository
        baseurl=https://anonymous:anonymous@public.dhe.ibm.com/aix/freeSoftware/aixtoolbox/RPMS/ppc/
        enabled=1
        gpgcheck=0
        [AIX_Toolbox_noarch]
        name=AIX noarch repository
        baseurl=https://anonymous:anonymous@public.dhe.ibm.com/aix/freeSoftware/aixtoolbox/RPMS/noarch/
        enabled=1
        gpgcheck=0
        [AIX_Toolbox_71]
        name=AIX 7.1 specific repository
        baseurl=https://anonymous:anonymous@public.dhe.ibm.com/aix/freeSoftware/aixtoolbox/RPMS/ppc-7.1/
        enabled=1
        gpgcheck=0

        [AIX_Toolbox_72]
        name=AIX 7.2 specific repository
        baseurl=https://anonymous:anonymous@public.dhe.ibm.com/aix/freeSoftware/aixtoolbox/RPMS/ppc-7.2/
        enabled=1
        gpgcheck=0
        
d.	If the service is behind proxy set proxy server in repository or export the https_proxy for internet access

        echo “proxy=http://user:password@proxyserver:port” >> /etc/yum.repos.d/aix.repo
        (or)
        export https_proxy= http://user:password@proxyserver:port

 e.	Verify the server is able to query AIX repositories
 
       -yum repolist
       Should return output like below
        repo id                                         repo name                                                              status
        !AIX_Toolbox                                    AIX generic repository                                                  2,508
        !AIX_Toolbox_71                                 AIX 7.1 specific repository                                                98
        !AIX_Toolbox_72                                 AIX 7.2 specific repository                                               170
        !AIX_Toolbox_noarch                             AIX noarch repository                                                     235
        
  So for we have configured AIX Toolbox repo in our repository server.
  
  # 2.	Synchronize and Create Local Repository
  
  a.	Execute reposync to sync AIX repositories to local directory
  
          #reposync -p <repo_dir> -r AIX_Toolbox -a ppc
          #reposync -p <repo_dir> -r AIX_Toolbox_61 -a ppc
          #reposync -p <repo_dir> -r AIX_Toolbox_71 -a ppc
          #reposync -p <repo_dir> -r AIX_Toolbox_72 -a ppc
          #reposync -p <repo_dir> -r AIX_Toolbox_noarch

b.	Verify the repositories are synced locally

        # ls -ltr <repo_dir>
        drwxr-xr-x. 253 root root 8192 Jun 23 09:52 AIX_Toolbox
        drwxr-xr-x.  11 root root  151 Jun 23 09:52 AIX_Toolbox_71
        drwxr-xr-x.  15 root root  211 Jun 23 09:52 AIX_Toolbox_72
        drwxr-xr-x.  45 root root 4096 Jun 23 09:50 AIX_Toolbox_noarch

c.	Create local repository to serve the AIX clients with the data synced from IBM repo

        # createrepo <repo_dir>/AIX_Toolbox
        # createrepo <repo_dir>/AIX_Toolbox_61
        # createrepo <repo_dir>/AIX_Toolbox_71
        # createrepo <repo_dir>/AIX_Toolbox_72
        # createrepo <repo_dir>/AIX_Toolbox_noarch
        
Now we have local repo ready in our repository server to serve AIX clients.

# 3.	Configure local repository to AIX clients through HTTP

a.	Check if httpd package is installed

      #rpm -qa httpd
      
b.	If not install httpd package

      #yum install httpd
      
c.	Configure the httpd server to serve local repository to AIX clients.
Add below lines in the end of /etc/httpd/conf/httpd.conf file in case of AIX (<IHS_install_dir>/conf/httpd.conf)

      <Directory "<repo_dir>">
          Options Indexes FollowSymLinks MultiViews
          AllowOverride None
          Require all granted
      </Directory>
      
d.	Start httpd services

      #systemctl start httpd
     	(or)
      # <IHS_install_dir>/bin/apachectl start (in case of AIX)

# 4.	Configure AIX Client to point private YUM repository 
a.	Edit the /opt/freeware/etc/yum/yum.conf file to add below entries in all AIX client servers

      [AIX_Toolbox]
      name=AIX generic repository
      baseurl=http://http_server_ip:80/repo_dir/AIX_Toolbox
      enabled=1
      gpgcheck=0

Sample /opt/freeware/etc/yum/yum.conf file.

      [main]
      cachedir=/var/cache/yum
      keepcache=1
      debuglevel=2
      logfile=/var/log/yum.log
      exactarch=1
      obsoletes=1
      plugins=1

      [AIX_Toolbox]
      name=AIX generic repository
      baseurl=http://http_server_ip:80/repo_dir/AIX_Toolbox
      enabled=1
      gpgcheck=0

      [AIX_Toolbox_noarch]
      name=AIX noarch repository
      baseurl=http://http_server_ip:80/repo_dir/AIX_Toolbox_noarch
      enabled=1
      gpgcheck=0

      [AIX_Toolbox_72]
      name=AIX 7.2 specific repository
      baseurl=http://http_server_ip:80/repo_dir/AIX_Toolbox_72
      enabled=1
      gpgcheck=0




