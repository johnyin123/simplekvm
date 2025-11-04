# simplekvm
- [x] Install
```
docker pull johnyinnews/simplekvm:trixie
docker pull johnyinnews/slapd:trixie
docker pull johnyinnews/etcd:trixie
```
- [x] Docker envs
```
META_SRV=simplekvm.registry.local
meta_srv_addr=192.168.167.1

# only simplekvm access GOLD_SRV
GOLD_SRV=simplekvm.registry.local
gold_srv_ipaddr=192.168.167.1

# tanent access CTRL_SRV, public access
CTRL_SRV=user.registry.local

LDAP_SRV=ldap
LDAP_PORT=10389
LDAP_PASSWORD='Pass4LDAP@docker'

ETCD_SRV=etcd
ETCD_PORT=2379
```
- [x] Install etcd
```
docker create --name ${ETCD_SRV} --restart always \
 --network br-int \
 --env ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379    \
 --env ETCD_ADVERTISE_CLIENT_URLS=http://0.0.0.0:2379 \
 --env ETCD_LOG_LEVEL='warn' \                    \
 johnyinnews/etcd:trixie
```
- [x] Install ldap
```
docker create --name ${LDAP_SRV} --restart always \
 --network br-int \
 --env LDAP_DOMAIN="myldap.internal" \
 --env LDAP_PASSWORD=${LDAP_PASSWORD} \
 johnyinnews/slapd:trixie
```
- [x] Install simplekvm
```
target=${simplekvm_dir} **config files dir** :octocat:
cat <<EOF
${target}/config
${target}/id_rsa
${target}/id_rsa.pub
${target}/ca.pem
${target}/client.key
${target}/client.pem
EOF
# envs: LDAP_UID_FMT, LDAP_BASE_DN, EXPIRE_SEC
# envs: DATA_DIR, ETCD_PREFIX, ETCD_SRV, ETCD_PORT, ETCD_CA, ETCD_KEY, ETCD_CERT, META_SRV, GOLD_SRV, CTRL_SRV, ACT_...
# --env DATA_DIR=/home/simplekvm/data -v ${target}/data:/home/simplekvm/data  \\
#  LDAP_SRV=ip
#  LDAP_PORT=port
# --env LDAP_UID_FMT='cn={uid}'
docker create --name simplekvm --restart always \
 --network br-int --ip 192.168.169.123 \
 --env LDAP_SRV_URL=ldap://${LDAP_SRV}:${LDAP_PORT} --env JWT_ALLOWS='["admin", "simplekvm"]' \
 --env META_SRV=${META_SRV} \
 --env CTRL_SRV=${CTRL_SRV} \
 --env GOLD_SRV=${GOLD_SRV} --add-host ${GOLD_SRV}:${gold_srv_ipaddr} \
 --env ETCD_PREFIX=/simple-kvm/work --env ETCD_SRV=${ETCD_SRV} --env ETCD_PORT=${ETCD_PORT} \
 -v ${target}/config:/home/simplekvm/.ssh/config \
 -v ${target}/id_rsa:/home/simplekvm/.ssh/id_rsa \
 -v ${target}/id_rsa.pub:/home/simplekvm/.ssh/id_rsa.pub \
 -v ${target}/ca.pem:/etc/pki/CA/cacert.pem \
 -v ${target}/client.key:/etc/pki/libvirt/private/clientkey.pem \
 -v ${target}/client.pem:/etc/pki/libvirt/clientcert.pem \
 -v ${target}/client.key:/etc/nginx/ssl/simplekvm.key \
 -v ${target}/client.pem:/etc/nginx/ssl/simplekvm.pem \
 johnyinnews/simplekvm:trixie
```
- [x] init ldap users
```
uid=test
LDAP_PASSWORD=password
# ####################
LDAP_PASSWORD_SSHA=$(docker exec -it ldap slappasswd -u -h '{SSHA}' -s ${LDAP_PASSWORD})
export LDAP_CFG_DIR="/etc/ldap/slapd.d"
cat<<EO_LDIF | tee log.txt | docker exec -i ldap su openldap -s /bin/bash -c "slapadd -n1 -F ${LDAP_CFG_DIR}"
dn: uid=${uid},ou=people,dc=myldap,dc=internal
objectClass: person
objectClass: shadowAccount
cn: ${uid}
sn: simplekvm ${uid}
userPassword: ${LDAP_PASSWORD_SSHA}
shadowMax: 60
shadowMin: 1
shadowWarning: 7
shadowInactive: 7
shadowLastChange: $(echo $(($(date "+%s")/60/60/24)))
EO_LDIF
```
![ui](https://github.com/user-attachments/assets/4cc3f93d-8ee8-4e7a-ad5b-80eec341ba78)



https://github.com/user-attachments/assets/276705f7-6299-410b-8290-a83422a55b1d

