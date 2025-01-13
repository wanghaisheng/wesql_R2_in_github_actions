Open R2.

Select Manage R2 API Tokens on the right top side of the dashboard.
Select Create API token.
Select the pencil icon or R2 Token text to edit your API token name.
Under Permissions, select Read or Edit for your token.
Select Create API Token.
Save a copy of your Access Key ID and Secret access key for the next step.


Here's the modified workflow with the necessary changes for Cloudflare R2, along with explanations:

```yaml
name: Start WeSQL Database

on:
  workflow_dispatch:

concurrency:
  group: start_wesql_database
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Configure AWS CLI for Cloudflare R2
        run: |
          aws configure set aws_access_key_id ${{ secrets.CLOUDFLARE_R2_ACCESS_KEY }}
          aws configure set aws_secret_access_key ${{ secrets.CLOUDFLARE_R2_SECRET_KEY }}
          aws configure set default.region auto
          aws configure set s3.endpoint_url https://${{ secrets.CLOUDFLARE_R2_ACCOUNT_ID }}.r2.cloudflarestorage.com
          aws configure set s3.signature_version s3v4

      - name: Start WeSQL Server
        run: |
          export WESQL_OBJECTSTORE_BUCKET=${{ secrets.CLOUDFLARE_R2_BUCKET_NAME }}
          export WESQL_OBJECTSTORE_REGION=auto #  Use auto for Cloudflare R2
          export WESQL_OBJECTSTORE_ACCESS_KEY=${{ secrets.CLOUDFLARE_R2_ACCESS_KEY }}
          export WESQL_OBJECTSTORE_SECRET_KEY=${{ secrets.CLOUDFLARE_R2_SECRET_KEY }}

          docker run -itd --network host --name wesql-server \
            -p 3306:3306 \
            -e MYSQL_CUSTOM_CONFIG="[mysqld]\n\
            port=3306\n\
            log-bin=binlog\n\
            gtid_mode=ON\n\
            enforce_gtid_consistency=ON\n\
            log_slave_updates=ON\n\
            binlog_format=ROW\n\
            objectstore_provider='aws'\n\
            repo_objectstore_id='tutorial'\n\
            objectstore_bucket='${WESQL_OBJECTSTORE_BUCKET}'\n\
            objectstore_region='${WESQL_OBJECTSTORE_REGION}'\n\
             objectstore_endpoint='https://${{ secrets.CLOUDFLARE_R2_ACCOUNT_ID }}.r2.cloudflarestorage.com'\n\
            branch_objectstore_id='main'" \
            -v ~/wesql-local-dir:/data/mysql \
            -e WESQL_CLUSTER_MEMBER='127.0.0.1:13306' \
            -e MYSQL_ROOT_PASSWORD=${{ secrets.WESQL_ROOT_PASSWORD }} \
            -e WESQL_OBJECTSTORE_ACCESS_KEY=${WESQL_OBJECTSTORE_ACCESS_KEY} \
            -e WESQL_OBJECTSTORE_SECRET_KEY=${WESQL_OBJECTSTORE_SECRET_KEY} \
            apecloud/wesql-server:8.0.35-0.1.0_beta3.38

      - name: Wait for MySQL port
        run: |
          for i in {1..60}; do
            if nc -z localhost 3306; then
              echo "MySQL port 3306 is ready!"
              exit 0
            fi
            echo "Waiting for MySQL port 3306..."
            sleep 5
          done
          echo "Timeout waiting for MySQL port 3306"
          exit 1

      - name: Start and parse Serveo tunnel
        run: |
          # Start Serveo SSH tunnel for MySQL (port 3306)
          nohup ssh -o StrictHostKeyChecking=no -R 0:localhost:3306 serveo.net > serveo.log 2>&1 &
          sleep 5

          # Parse the tunnel information
          TUNNEL_LINE=$(grep 'Forwarding TCP' serveo.log || true)
          if [ -z "$TUNNEL_LINE" ]; then
            echo "No forwarding line found in serveo.log"
            exit 1
          fi

          HOST="serveo.net"
          PORT=$(echo "$TUNNEL_LINE" | tr -d '\r' | sed 's/[[:space:]]*$//' | grep -oE '[0-9]+$' || true)

          if [ -z "$PORT" ]; then
            echo "No port found in the forwarding line"
            exit 1
          fi

          echo "MySQL Public Access:"
          echo "Host: $HOST"
          echo "Port: $PORT"
          echo "Connect: mysql -h $HOST -P $PORT -u root -p"

          # Export HOST and PORT for the next step
          echo "HOST=$HOST" >> $GITHUB_ENV
          echo "PORT=$PORT" >> $GITHUB_ENV

      - name: Write Connection Info to S3
        run: |
          cat << EOF > connection_info.txt
          host=$HOST
          port=$PORT
          username=root
          password=${{ secrets.WESQL_ROOT_PASSWORD }}
          mysql_cli=mysql -h $HOST -P $PORT -u root -p${{ secrets.WESQL_ROOT_PASSWORD }}
          EOF
          
          aws s3 cp connection_info.txt s3://${{ secrets.CLOUDFLARE_R2_BUCKET_NAME }}/connection_info.txt
          echo "Connection properties written to s3://${{ secrets.CLOUDFLARE_R2_BUCKET_NAME }}/connection_info.txt"

      - name: Keep session running
        run: |
          echo "Keep session running"
          echo "You can review the connection info in the S3 bucket by reading the file connection_info.txt"
          echo "aws s3 cp s3://${{ secrets.CLOUDFLARE_R2_BUCKET_NAME }}/connection_info.txt -"
          
          tail -f /dev/null
```

**Key Changes and Explanations:**

1.  **Secret Names:**
    *   I've renamed the secrets to start with `CLOUDFLARE_R2_` to avoid confusion and make it explicit we are using cloudflare R2 (e.g., `CLOUDFLARE_R2_ACCESS_KEY`, `CLOUDFLARE_R2_SECRET_KEY`, `CLOUDFLARE_R2_BUCKET_NAME`, `CLOUDFLARE_R2_ACCOUNT_ID`). Please ensure that these are named correctly in your GitHub Action's secrets
2.  **AWS CLI Configuration for R2:**
    *   `aws configure set aws_access_key_id ${{ secrets.CLOUDFLARE_R2_ACCESS_KEY }}`:  Sets the access key ID for Cloudflare R2.
    *   `aws configure set aws_secret_access_key ${{ secrets.CLOUDFLARE_R2_SECRET_KEY }}`: Sets the secret access key for Cloudflare R2.
    *   `aws configure set default.region auto`: Sets the default region to `auto`, as Cloudflare R2 handles routing automatically.
    *   `aws configure set s3.endpoint_url https://${{ secrets.CLOUDFLARE_R2_ACCOUNT_ID }}.r2.cloudflarestorage.com`:  Sets the correct endpoint for Cloudflare R2, using the provided account ID.
    *   `aws configure set s3.signature_version s3v4`: Sets the signature version to `s3v4` which is required for R2.
3.  **WeSQL Server Environment Variables:**
    *   `export WESQL_OBJECTSTORE_BUCKET=${{ secrets.CLOUDFLARE_R2_BUCKET_NAME }}`: Uses your R2 bucket name.
    *   `export WESQL_OBJECTSTORE_REGION=auto`:  Sets region to `auto`.
    *   `objectstore_endpoint='https://${{ secrets.CLOUDFLARE_R2_ACCOUNT_ID }}.r2.cloudflarestorage.com'`:  Sets R2 endpoint in the MYSQL_CUSTOM_CONFIG.
4.  **Connection Info to S3:** The `Write Connection Info to S3` will upload connection information to R2 using the cloudflare config. The bucket location will be in `${{ secrets.CLOUDFLARE_R2_BUCKET_NAME }}`.

**Environment Variables:**

To use this workflow, you will need to define the following secrets in your GitHub repository settings:

*   **`CLOUDFLARE_R2_ACCESS_KEY`:** The access key for your Cloudflare R2 bucket.
*   **`CLOUDFLARE_R2_SECRET_KEY`:** The secret key for your Cloudflare R2 bucket.
*   **`CLOUDFLARE_R2_BUCKET_NAME`:** The name of the R2 bucket you are using.
*   **`CLOUDFLARE_R2_ACCOUNT_ID`:** Your Cloudflare account ID.
*   **`WESQL_ROOT_PASSWORD`:** The root password for the WeSQL server.

**How to Use:**

1.  **Configure Secrets**: Make sure your repository secrets are set up correctly as mentioned above.
2.  **Trigger Workflow:** Trigger the `Start WeSQL Database` workflow using a `workflow_dispatch` event in GitHub Actions.
3.  **Review Output:** You will see the public host and port to connect to the database in the action's output. The connection info will also be in the R2 bucket.

**Additional Notes:**

*   **Endpoint:** Make sure the endpoint you are using in the `aws configure` command is correct for Cloudflare R2.
*   **Permissions:**  Make sure the API key you are using for Cloudflare has the correct permission to read/write to your bucket.
*   **WeSQL Documentation:**  Refer to the WeSQL server documentation for details about how it interacts with object storage.
* **Region:** Cloudflare R2 doesn't require a region, so it's set to `auto`.

This updated workflow will configure the AWS CLI and WeSQL to use Cloudflare R2, making it more convenient to use Cloudflare as your object storage solution. Let me know if you have further questions or run into any issues.
