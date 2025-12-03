```sh
[root@exadevdb-qwrny1 ~]# dbaascli cswLib listLocal --product GRID
DBAAS CLI version 25.3.1.0.0
Executing command cswLib listLocal --product GRID
Job id: 6d233084-c85e-4074-9c10-63e97e265b4d
Session log: /var/opt/oracle/log/cswLib/listLocal/dbaastools_2025-12-02_04-59-56-PM_271928.log
Log file location: /var/opt/oracle/log/cswLib/listLocal/dbaastools_2025-12-02_04-59-56-PM_271928.log

############ List of Available Grid Images  #############


dbaascli execution completed



[root@exadevdb-qwrny1 ~]# dbaascli cswlib download --IMAGETAG grid_19.29.0.0.0
DBAAS CLI version 25.3.1.0.0
Executing command cswlib download --IMAGETAG grid_19.29.0.0.0
Job id: f22e3937-7f8f-4be5-8cf5-49c1f4e2ab63
Session log: /var/opt/oracle/log/cswLib/download/dbaastools_2025-12-02_05-06-31-PM_307511.log
Loading PILOT...
Session ID of the current execution is: 38420
Log file location: /var/opt/oracle/log/cswLib/download/pilot_2025-12-02_05-09-13-PM_320740
-----------------
Running initialize job
Completed initialize job
-----------------
Running validate_url_reachability job
Completed validate_url_reachability job
-----------------
Running validate_image_tag_existence job
Completed validate_image_tag_existence job
-----------------
Running validate_free_space job
Completed validate_free_space job
-----------------
Running validate_file_permissions job
Completed validate_file_permissions job
-----------------
Acquiring write lock: grid_19.29.0.0.0
Running pre_download_lock_manager job
Completed pre_download_lock_manager job
-----------------
Running download_image job
Image location=/var/opt/oracle/dbaas_acfs/gi_images/grid_19.29.0.0.0
Download succeeded
Completed download_image job
-----------------
Running validate_sha256sum job
Skipping Sha256Sum check because the catalog does not contain sha256sum
Completed validate_sha256sum job
-----------------
Running decrypt job
Completed decrypt job
-----------------
Running post_download_update job
Completed post_download_update job
-----------------
Running verify_signature job
Completed verify_signature job
-----------------
Running update_file_permissions job
Completed update_file_permissions job
-----------------
Running post_download_lock_manager job
Completed post_download_lock_manager job
Releasing lock: grid_19.29.0.0.0
{"grid_19.29.0.0.0":{"file_list":[{"file":"/var/opt/oracle/dbaas_acfs/gi_images/grid_19.29.0.0.0/grid_19.29.0.0_Image.zip"}]}}

dbaascli execution completed



[root@exadevdb-qwrny1 ~]# dbaascli cswLib listLocal --product GRID
DBAAS CLI version 25.3.1.0.0
Executing command cswLib listLocal --product GRID
Job id: 23ae9982-fce1-4d74-8a87-b675b15fcc24
Session log: /var/opt/oracle/log/cswLib/listLocal/dbaastools_2025-12-02_05-56-46-PM_207322.log
Log file location: /var/opt/oracle/log/cswLib/listLocal/dbaastools_2025-12-02_05-56-46-PM_207322.log

############ List of Available Grid Images  #############

1.IMAGE_TAG=grid_19.29.0.0.0
  IMAGE_SIZE=5GB
  VERSION=19.29.0.0.0

```
  DESCRIPTION=19c OCT 2025 GI Image

dbaascli execution completed
