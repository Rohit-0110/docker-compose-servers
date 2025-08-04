```bash
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --mysql > initdb.sql

docker cp initdb.sql guacamoledb:/initdb.sql

docker exec -it guacamoledb bash

cat /initdb.sql | mysql -u root -p guacamole_db

Enter the MariaDB root password as set above
exit

```