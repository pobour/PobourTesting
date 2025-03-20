#!/bin/bash

# Variables
SONARQUBE_VERSION="10.3.0.82913"
SONARQUBE_USER="sonarqube"
SONARQUBE_HOME="/opt/sonarqube"
POSTGRES_DB="sonarqube"
POSTGRES_USER="sonar"
POSTGRES_PASSWORD="SonarQube@123"

# Update the system
echo "Updating system packages..."
sudo yum update -y

# Install required dependencies
echo "Installing required packages..."
sudo yum install -y epel-release wget unzip git java-17-openjdk java-17-openjdk-devel

# Install PostgreSQL (SonarQube requires a database)
echo "Installing PostgreSQL..."
sudo yum install -y postgresql-server postgresql-contrib

# Initialize and start PostgreSQL
echo "Initializing PostgreSQL database..."
sudo postgresql-setup --initdb --unit postgresql
sudo systemctl enable --now postgresql

# Configure PostgreSQL
echo "Configuring PostgreSQL..."
sudo -u postgres psql -c "CREATE DATABASE $POSTGRES_DB;"
sudo -u postgres psql -c "CREATE USER $POSTGRES_USER WITH ENCRYPTED PASSWORD '$POSTGRES_PASSWORD';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE $POSTGRES_DB TO $POSTGRES_USER;"

# Update PostgreSQL settings
echo "Updating PostgreSQL configuration..."
sudo bash -c 'echo "host all all 0.0.0.0/0 md5" >> /var/lib/pgsql/data/pg_hba.conf'
sudo bash -c 'echo "listen_addresses = '\''*'\''" >> /var/lib/pgsql/data/postgresql.conf'

# Restart PostgreSQL
echo "Restarting PostgreSQL..."
sudo systemctl restart postgresql

# Create SonarQube user
echo "Creating SonarQube user..."
sudo useradd -m -d $SONARQUBE_HOME -s /bin/bash $SONARQUBE_USER

# Download and extract SonarQube
echo "Downloading SonarQube..."
cd /opt
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-$SONARQUBE_VERSION.zip
sudo unzip sonarqube-$SONARQUBE_VERSION.zip
sudo mv sonarqube-$SONARQUBE_VERSION sonarqube
sudo rm -f sonarqube-$SONARQUBE_VERSION.zip

# Set permissions
echo "Setting permissions..."
sudo chown -R $SONARQUBE_USER:$SONARQUBE_USER $SONARQUBE_HOME

# Configure SonarQube
echo "Configuring SonarQube..."
sudo bash -c "cat > $SONARQUBE_HOME/conf/sonar.properties <<EOF
sonar.jdbc.username=$POSTGRES_USER
sonar.jdbc.password=$POSTGRES_PASSWORD
sonar.jdbc.url=jdbc:postgresql://localhost:5432/$POSTGRES_DB

sonar.web.host=0.0.0.0
sonar.web.port=9000
sonar.web.javaAdditionalOpts=-server
sonar.log.level=INFO
sonar.path.logs=logs
EOF"

# Create systemd service file
echo "Creating systemd service file for SonarQube..."
sudo bash -c 'cat > /etc/systemd/system/sonarqube.service <<EOF
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonarqube
Group=sonarqube
Restart=always
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
EOF'

# Reload systemd and start SonarQube
echo "Starting SonarQube..."
sudo systemctl daemon-reload
sudo systemctl enable --now sonarqube

# Check status
echo "SonarQube service status:"
sudo systemctl status sonarqube --no-pager
