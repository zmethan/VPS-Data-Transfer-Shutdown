#!/bin/bash

# Check if enough parameters are provided
if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <interface_name> <traffic_limit>"
    exit 1
fi

# Parameters
interface_name=$1
traffic_limit=$2

# Update package list and install cron service
sudo apt update
sudo apt install cron -y

# Install dependencies
sudo apt install vnstat bc -y

# Configure vnstat
sudo sed -i '0,/^;Interface ""/s//Interface '\"$interface_name\"'/' /etc/vnstat.conf
sudo sed -i "0,/^;UnitMode.*/s//UnitMode 1/" /etc/vnstat.conf
sudo sed -i "0,/^;MonthRotate.*/s//MonthRotate 1/" /etc/vnstat.conf

# Restart vnstat service
sudo systemctl restart vnstat

# Create automatic shutdown script check.sh
cat << EOF | sudo tee /root/check.sh > /dev/null
#!/bin/bash

# Network interface name
interface_name="$interface_name"
# Traffic threshold limit (in GB)
traffic_limit=$traffic_limit

# Update network interface record
vnstat -i "$interface_name"

# Get monthly usage, \$11: total traffic (in+out); \$10: outgoing; \$9: incoming
TRAFF_USED=\$(vnstat --oneline b | awk -F';' '{print \$11}')

# Check if data was retrieved successfully
if [[ -z "\$TRAFF_USED" ]]; then
    echo "Error: Not enough data available yet."
    exit 1
fi

# Convert traffic to GB
CHANGE_TO_GB=\$(echo "scale=2; \$TRAFF_USED / 1073741824" | bc)

# Verify that the converted traffic value is a valid number
if ! [[ "\$CHANGE_TO_GB" =~ ^[0-9]+([.][0-9]+)?\$ ]]; then
    echo "Error: Invalid traffic data."
    exit 1
fi

# Compare if the traffic exceeds the threshold
if (( \$(echo "\$CHANGE_TO_GB > \$traffic_limit" | bc -l) )); then
    sudo /usr/sbin/shutdown -h now
fi
EOF

# Grant execute permission
sudo chmod +x /root/check.sh

# Set a cron job to check every 3 minutes
(crontab -l ; echo "*/3 * * * * /bin/bash /root/check.sh > /root/shutdown_debug.log 2>&1") | crontab -

echo "All done! The script has been installed and configured successfully."
