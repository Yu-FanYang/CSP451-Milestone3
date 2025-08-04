# CSP451-Milestone3

## Objective
- Design a product stock events emitted by backend and consumed asynchronously by Azure Function while routed to Supplier API microservice
- Run Docker container on VM
- Trace full message flow with logging and correlation IDs


### Event type chosen and why (Queue/Service Bus/Event Grid)
I used Azure Storage Queue to finish this project becasue it allows me to efficiently connect to Azure Function with connection string. After it's connected to Azure Function, Function will then connected to VM which is Supplier-api and will notify when to restock inventory. 

### Message format and flow
- Message format from backend: *product: ${productName}
  Stock: ${currentStock}
  Correlation ID: ${correlationId}*



### Create a Supplier API Microservice

1. Create Storage Account and Storage Queue
```
$RGN="csp451-yyang334"
$LOC="eastasia"
$SAN="yyang334storage"
az storage account create --name $SAN --resource-group $RGN --location $LOC --sku Standard_LRS --allow-blob-public-access false
az storage queue create --name product-stock-events --account-name $SAN
```

- Output connection string by running the following:
`az storage account show-connection-string --name $SAN --resource-group $RGN --query 'connectionString' --output tsv`


2. Create a VM on Azure portal 
- Get the Public IP and SSH into VM
- Generate ssh key from local computer in order to upload the local files to VM: `ssh-keygen -t rsa -b 4096`
  
3. Create a new Dockerized supplier-api with the following scripts:

- App.js
```
const express = require('express');
const app = express();
const port = 3000; 
app.use(express.json()); 
app.post('/order', (req, res) => { 
  const correlationId = req.body.correlationId || 'N/A'; 
  const product = req.body.product || 'Unknown Product';
  const quantity = req.body.quantity || 0;
  console.log(`[Supplier API] Received order for ${product} (Quantity: ${quantity}) with Correlation ID: ${correlationId}`); 
  res.json({ message: `Order for ${product} received successfully!`, correlationId: correlationId, status: 'confirmed' }); 
});
app.listen(port, () => {
  console.log(`Supplier API listening at http://localhost:${port}`);
});
```

- Dockerfile
```
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./ 
RUN npm install 
COPY . . 
EXPOSE 3000 
CMD ["node", "app.js"]
```

- docker-compose.yml
```
version: '3.8' 
services:
  supplier-api: 
    build: ./supplier-api 
    ports:
      - "3000:3000" 
    restart: always
```


4. Deploy to the same Azure VM via docker-compose
- Run the following commands to install docker-compose:
```
  sudo apt-get update
  sudo apt-get install ca-certificates curl gnupg lsb-release
  sudo mkdir -m 0755 -p /etc/apt/keyrings
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  echo \ "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  sudo apt-get update
  sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
  sudo usermod -aG docker azureuser
```
5. Then exit the VM and reconnect

6. Confirm docker version
`docker --version`
`docker compose version`

7.	Upload the supplier-api and docker-compose files from local machine to VM
Use the SSH generated earlier
- Copy supplier-api folder to VM: `scp -r ./supplier-api azureuser@57.158.26.57:/home/azureuser/`
- Copy docker-compose.yml to VM: `scp ./docker-compose.yml azureuser@57.158.26.57:/home/azureuser/`

8.	Start Docker service

### Create an Azure Function Subscriber
Create an Azure Function with a trigger
- Create Function App
```
$RGN="csp451-yyang334"
$LOC="eastasia"
$SAN="yyang334storage"
$FAPPN="yyang334app"
az functionapp create --resource-group $RGN --consumption-plan-location $LOC --name $FAPPN --storage-account $SAN --runtime node --runtime-version 22 --functions-version 4
```
Created Queue trigger
- func init --worker-runtime node --model V4 --docker
- func new --name SimpleQueueProcessor --template AzureQueueStorageTrigger --language JavaScript

### Enable Tracceability and Log Output
1.	Use Azure Monitor to trace end-to-end flow
- Enable Azure Monitor traceability from Azure portal
- Azure portal> Monitor > Virtual Machine > Enable

2.	Trace log output from Function app
- From Function app > Log Stream > shows “Connected” to make sure Function app is working
- Run `node index.js` on local machine to trigger function


3.	Check log output from Backend, Function App, and Supplier API with correlation ID
- Please refer to the documents for the cooresponding screenshots for log output from Backend, Function App, and Supplier API


### Source code

- Appendix A: Backend service source code
- Path: `C:\yyang334\smartretail-project\backend\index.js`

``` javascript
const { QueueClient } = require("@azure/storage-queue"); 
const { v4: uuidv4 } = require('uuid'); 
const connectionString = "XXX";
const queueName = "product-stock-events"; 
const queueClient = new QueueClient(connectionString, queueName);
async function emitProductStockEvent(productName, currentStock) {
  const threshold = 10; 
  if (currentStock <= threshold) {
    const correlationId = uuidv4(); 
    const eventData = {
      product: productName,
      currentStock: currentStock,
      message: `Product ${productName} stock is low (${currentStock}). Please reorder.`,
      correlationId: correlationId 
    };
    const message = Buffer.from(JSON.stringify(eventData)).toString('base64'); 

    try {
      await queueClient.sendMessage(message); 
      console.log(`[Backend] Emitted stock event for ${productName} (Stock: ${currentStock}). Correlation ID: ${correlationId}`);  
    } catch (error) {
      console.error(`[Backend] Error emitting event for ${productName}:`, error.message);
    }
  } else {
    console.log(`[Backend] Product ${productName} stock is fine (${currentStock}). No event emitted.`);
  }
}
async function simulateStockDecrease() {
  await emitProductStockEvent("Laptop", 5); 
  await emitProductStockEvent("Mouse", 25); 
  await emitProductStockEvent("Keyboard", 8); 
}
simulateStockDecrease();
```

- Appendix B: Supplier API microservice source code
- Path: `C:\yyang334\smartretail-project\supplier-api\app.js`

``` javascript
const express = require('express');
const app = express();
const port = 3000; 
app.use(express.json()); 

app.post('/order', (req, res) => { 
  const correlationId = req.body.correlationId || 'N/A'; 
  const product = req.body.product || 'Unknown Product';
  const quantity = req.body.quantity || 0;
  console.log(`[Supplier API] Received order for ${product} (Quantity: ${quantity}) with Correlation ID: ${correlationId}`); 
  res.json({ message: `Order for ${product} received successfully!`, correlationId: correlationId, status: 'confirmed' }); 
});
app.listen(port, () => {
  console.log(`Supplier API listening at http://localhost:${port}`);
});
```

- Appendix C: Azure Function trigger source code
- Path: `C:\yyang334\smartretail-project\test-function-app\src\functions\SimpleQueueProcessor.js`

``` javascript
const { app } = require('@azure/functions');
const axios = require('axios');
app.storageQueue('SimpleQueueProcessor', {
    //queueName: 'js-queue-items',
    queueName: 'product-stock-events',
    connection: 'AzureWebJobsStorageConnection',
    handler: async (message, context) => { 
            context.log(`[Azure Function] Queue trigger function processed message: ${JSON.stringify(message)}`); 
    
            const payload = message; 
            const correlationId = payload.correlationId 
            const product = payload.product;
            const currentStock = payload.currentStock;
            context.log(`[Azure Function] Processing product: ${product}, Stock: ${currentStock}, Correlation ID: ${correlationId}`);
            try {
                //const supplierApiUrl = 'http://localhost:3000/order'; 
                const supplierApiUrl = 'http://57.158.26.57:3000/order'; 
                const response = await axios.post(supplierApiUrl, { 
                    product: product,
                    quantity: currentStock, 
                    correlationId: correlationId
                });
                context.log(`[Azure Function] Supplier API Response for Correlation ID ${correlationId}: ${JSON.stringify(response.data)}`); 
            } catch (error) {
                context.error(`[Azure Function] Error calling Supplier API for Correlation ID ${correlationId}: ${error.message}`); 
            }
        }
});
```
