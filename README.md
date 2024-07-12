# Typesafe(Swagger) Client Generator
## Introduction
The main purpose of this project is to achieve type safety between the frontend and backend. Often, backend developers make changes to types or introduce new variables without informing the frontend developers, resulting in crashes when the frontend runs due to type mismatches. This project ensures type safety by automatically generating and uploading a new schema whenever backend code is committed. On the frontend, this new schema is fetched before building or committing code. If there are any changes, build errors will occur, allowing developers to fix the code before it crashes the program.

### Key features:
1. Automated schema uploading and fetching
2. Branch-specific schema handling
3. Hash-based schema comparison to reduce redundant operations
4. Middleware and interceptors for request handling

## Table of Contents
1. [Installation](#installation)
2. [Usage](#usage)
3. [Server Side](#server-side)
4. [Client Side](#client-side)

## Installation
### Frontend
1. Clone the repository:
    ```bash
    git clone https://github.com/AliSaif541/example-app-openapi-fetch.git
    cd example-app-openapi-fetch
    ```

2. Install the dependencies:
    ```bash
    npm install
    ```
3. Set up environment variables

### Backend
1. Clone the repository:
    ```bash
    git clone https://github.com/AliSaif541/example-server-tsoa.git
    cd example-server-tsoa
    ```

2. Install the dependencies:
    ```bash
    npm install
    ```
3. Set up environment variables

## Usage
### Frontend
1. Create route functions to interact with the backend API.
2. Fetch the schema from Google Drive and convert it to Typescript interfaces:
    ```bash
    npm run fetch-swagger
    ```
3. Start the client
    ```bash
    npm run dev
    ```
4. Build the code:
    ```bash
    npm run build
    ```

### Backend
1. Create controller functions using `tsoa` decorators.
2. Generate routes and schema files:
    ```bash
    npm run generate-swagger
    ```
3. Upload to Google Drive:
    ```bash
    npm run upload-swagger
    ```
4. Start the server:
    ```bash
    npm run dev
    ```
5. Build the code:
    ```bash
    npm run build
    ```

## Server Side
The server-side of this project is designed to ensure type safety and seamless integration with the frontend. By generating routes and schemas using `tsoa`, creating middlewares for security, uploading schema files to Google Drive, and automating these processes, we can maintain consistent and reliable type definitions across the project.

### Key Components
1. Creating Routes through tsoa
2. Creating Middlewares using tsoa
3. Uploading the Schema to Google Drive
4. Automating Schema Storage with Husky

### Creating Routes through tsoa
Using tsoa, we define controllers and routes with decorators to handle API requests. This ensures that the routes are well-documented and consistent with the generated OpenAPI schema. By leveraging tsoa, we can maintain a single source of truth for our API definitions, automatically generating OpenAPI specifications based on the annotations in our controllers. This approach significantly reduces the chances of discrepancies between the implementation and the API documentation, thereby enhancing type safety and consistency across the application.

Tsoa decorators like `@Route`, `@Get`, `@Post`, `@SuccessResponse`, `@Query`, `@Body`, and `@Res` are used to define the HTTP methods, paths, expected responses, query parameters, request bodies, and response handlers for each route. These annotations make the code self-documenting and facilitate the generation of OpenAPI specifications.

Below is an example of how to create routes using tsoa with various decorators:

```ts
@Route("/petsapp/pet")
export class PetsController extends Controller {
    @SuccessResponse("200", "Found")
    @Get("/single")
    @Middlewares(authMiddleware)
    public async getAPet(@Res() success: TsoaResponse<200, IPet>, @Res() error: TsoaResponse<500, { status: string, message: string }>, @Query() name: string): Promise<void> {
        try {
            const getAPetDTO = GetAPetDTO.createDTO({ name });
            const petService = new petsService();
            const httpResponse = await petService.getAPet(getAPetDTO);

            success(200, httpResponse.body as IPet);
        } catch (err) {
            error(500, { status: "error", message: (err as Error).message });
        }
    }
}
```

### Creating Middlewares using tsoa
Middlewares are used to perform tasks such as authentication, logging, validation, or modifying the request and response objects. In tsoa, middlewares are used to add additional layers of functionality to your API endpoints.

In this example, we implement an authentication middleware to protect our API routes. The middleware checks for a specific token in the Authorization header of the request. If the token matches the expected value, the request is allowed to proceed; otherwise, a 401 Unauthorized response is sent.

```ts
export function authMiddleware(req: Request, res: Response, next: NextFunction) {
    const token = req.headers["authorization"];

    if (token === TOKEN) {
        next();
    } else {
        res.status(401).json({ status: "error", message: "Unauthorized" });
    }
}
```

### Uploading the Schema to Google Drive

To maintain type safety between the frontend and backend, it is crucial to have a consistent schema that both sides use. By uploading the OpenAPI schema to Google Drive, we ensure that the latest schema is always available for the frontend to fetch. This process involves several steps to manage the schema effectively and avoid redundant operations.

1. **Using Google Drive API**: The Google Drive API allows us to programmatically interact with Google Drive. This API provides methods to upload, list, and manage files in Google Drive. By leveraging this API, we can automate the process of uploading the schema file from our backend.

    In order to use it, we first have to set it up using OAuth2 credentials in order to authenticate with Google Drive. This involves providing a `credentials.json` file containing the necessary API keys. In addition, we also have to create a `drive.files.create` request to upload the file to a specified folder in Google Drive.

    ```ts
    import { google } from 'googleapis';

    const auth = new google.auth.GoogleAuth({
      keyFile: './credentials.json',
      scopes: ['https://www.googleapis.com/auth/drive'],
    });

    const drive = google.drive({ version: 'v3', auth });

    const fileMetadata = {
      name: 'schema.json',
      parents: [process.env.GOOGLE_DRIVE_FOLDER_ID],
    };
    const media = {
      mimeType: 'application/json',
      body: fs.createReadStream('path/to/schema.json'),
    };

    const response = await drive.files.create({
      requestBody: fileMetadata,
      media: media,
      fields: 'id',
    });
    console.log('Uploaded new file with Id:', response.data.id);
    ```

2. **Attaching Branch Names**: Different branches may have different versions of the schema. By attaching the branch name to the file name, we ensure that schemas specific to each branch are stored and retrieved accurately. This is done by first obtaining the current branch name using Git commands and then appending the branch name to the schema file name to differentiate between schemas from various branches.

    ```ts
    const branchName = execSync('git rev-parse --abbrev-ref HEAD').toString().trim();
    const fileName = `schema-${branchName}.json`;

    const fileMetadata = {
      name: fileName,
      parents: [process.env.GOOGLE_DRIVE_FOLDER_ID],
    };
    ```

3. **Comparing with Hashes**: In order to prevent redundant uploads, we compare the MD5 checksum of the local and remote files to avoid uploading the schema file if it has not changed. This reduces unnecessary operations and bandwidth usage and ensures that only modified schema files are uploaded.

    First, compute the checksum of the local file to identify its unique hash. Then, fetch the checksum of the existing file on Google Drive and compare them. If they don't match, an upload is needed.

    ```ts
    async function getFileChecksum(filePath: string): Promise<string> {
        const fileBuffer = await fsp.readFile(filePath);
        const hash = crypto.createHash('md5');
        hash.update(fileBuffer);
        return hash.digest('hex');
    }

    const localFileChecksum = await getFileChecksum('path/to/schema.json');
    // Compare with remote file checksum retrieved from Google Drive
    ```

4. **Attaching Hashes to Names**: Attaching a unique hash to the file name helps in managing different versions of the schema file. This prevents name collisions and ensures that each version of the file is distinct.

    ```ts
    const randomHash = crypto.randomBytes(6).toString('hex');
    const fileName = `schema-${branchName}-${randomHash}.json`;
    ```

5. **Abstracting the Upload Process**: To make the schema upload process more flexible and maintainable, an abstract approach is employed. This allows for easy integration with different storage services without modifying the core upload logic. 

    Define an abstract class for storage services:

    ```ts
    export abstract class StorageService {
      abstract listFiles(query: string): Promise<{ id: string, name: string, md5Checksum: string }[]>;
      abstract updateFile(fileId: string, newFileName: string, filePath: string): Promise<void>;
      abstract createFile(fileName: string, filePath: string): Promise<string>;
    }
    ```

    Implement the `StorageService` interface for Google Drive:

    ```ts
    import { google } from 'googleapis';
    import fs from 'fs';
    import { StorageService } from './storageService';

    export class GoogleDriveService extends StorageService {
      private drive;

      constructor() {
        super();
        const auth = new google.auth.GoogleAuth({
          keyFile: './credentials.json',
          scopes: ['https://www.googleapis.com/auth/drive'],
        });
        this.drive = google.drive({ version: 'v3', auth });
      }

      async listFiles(query: string): Promise<{ id: string, name: string, md5Checksum: string }[]> {
        const listResponse = await this.drive.files.list({
          q: query,
          fields: 'files(id, name, md5Checksum)',
        });

        return listResponse.data.files?.map(file => ({
          id: file.id!,
          name: file.name!,
          md5Checksum: file.md5Checksum!,
        })) || [];
      }

      async updateFile(fileId: string, newFileName: string, filePath: string): Promise<void> {
        await this.drive.files.update({
          fileId: fileId,
          requestBody: {
            name: newFileName,
          },
          media: {
            mimeType: 'application/json',
            body: fs.createReadStream(filePath),
          },
        });
      }

      async createFile(fileName: string, filePath: string): Promise<string> {
        const fileMetadata = {
          name: fileName,
          parents: [process.env.GOOGLE_DRIVE_FOLDER_ID!],
        };
        const media = {
          mimeType: 'application/json',
          body: fs.createReadStream(filePath),
        };

        const response = await this.drive.files.create({
          requestBody: fileMetadata,
          media: media,
          fields: 'id',
        });

        return response.data.id!;
      }
    }
    ```

    Define the abstract class `UploadProcess` and implement the `SchemaUploadProcess` class:

    ```ts
    import { StorageService } from '../services/storageService';

    export abstract class UploadProcess {
      protected storageService: StorageService;

      constructor(storageService: StorageService) {
        this.storageService = storageService;
      }

      abstract uploadFile(): Promise<void>;
    }

    import { execSync } from 'child_process';
    import crypto from 'crypto';
    import fs from 'fs';
    import fsp from 'fs/promises';
    import dotenv from 'dotenv';
    import { UploadProcess } from './uploadProcess';
    import { StorageService } from '../services/storageService';

    dotenv.config();

    export class SchemaUploadProcess extends UploadProcess {
      constructor(storageService: StorageService) {
        super(storageService);
      }

      private async getFileChecksum(filePath: string): Promise<string> {
        const fileBuffer = await fsp.readFile(filePath);
        const hash = crypto.createHash('md5');
        hash.update(fileBuffer);
        return hash.digest('hex');
      }

      async uploadFile(): Promise<void> {
        const branchName = execSync('git rev-parse --abbrev-ref HEAD').toString().trim();
        const localFilePath = 'http/output/swagger.json';

        try {
          const files = await this.storageService.listFiles(`name contains '${branchName}-' and trashed=false`);

          if (files.length > 0) {
            const { id: remoteFileId, name: remoteFileName, md5Checksum: remoteFileChecksum } = files[0];
            const localFileChecksum = await this.getFileChecksum(localFilePath);

            if (localFileChecksum === remoteFileChecksum) {
              console.log(`File ${remoteFileName} is the same as the local file. Skipping upload.`);
              return;
            }

            const randomHash = crypto.randomBytes(6).toString('hex');
            const newFileName = `schema-${branchName}-${randomHash}.json`;

            await this.storageService.updateFile(remoteFileId, newFileName, localFilePath);
            console.log('Updated file with Id:', remoteFileId);
          } else {
            const randomHash = crypto.randomBytes(6).toString('hex');
            const fileName = `schema-${branchName}-${randomHash}.json`;

            const fileId = await this.storageService.createFile(fileName, localFilePath);
            console.log('Uploaded new file with Id:', fileId);
          }

        } catch (error) {
          console.error('Error:', error);
        }
      }
    }
    ```

    **Usage Example**:

    To use the schema upload process with Google Drive:

    ```ts
    import { GoogleDriveService } from './services/googleDriveService';
    import { SchemaUploadProcess } from './processes/schemaUpload';

    async function implementingUpload() {
      const storageService = new GoogleDriveService();
      const uploadProcess = new SchemaUploadProcess(storageService);

      await uploadProcess.uploadFile();
    }

    implementingUpload();
    ```

By adopting this approach, the schema upload process remains flexible and can easily adapt to different storage services as needed. This approach not only maintains type safety across the frontend and backend but also optimizes resource usage and improves the overall development workflow.

### Automating Schema Storage with Husky
We use Husky to automate the process of storing the schema whenever code is committed or built. Husky is a tool that allows us to easily manage Git hooks. By adding a script to run the schema upload process in the pre-commit or pre-push hook, we ensure that the schema in Google Drive is always up-to-date.
1. Install Husky:
    ```bash
    npm install husky --save-dev
    ```
2. Enable Husky Hooks:

    ```bash
    npx husky install
    ```
3. Create Pre-Commit Hook:

    ```bash
    npx husky add .husky/pre-commit "npm run swagger"
    ```
4. Update package.json:

    ```json
    "scripts": {
        "prepare": "husky install",
        "generate-swagger": "npx tsoa routes && npx tsoa spec",
        "upload-swagger": "tsx scripts/storingSchema.ts",
        "swagger": "npm run generate-swagger && npm run upload-swagger",
    }
    ```

## Client Side
On the client side, the primary goal is to interact with the backend APIs efficiently and ensure type safety by leveraging the OpenAPI schema. This is achieved through several key components:

1. Making Route Functions for Sending Requests
2. Implementing Interceptors
3. Fetching and Managing the Schema
4. Automating Schema Updates with Husky

### Making Route Functions for Sending Requests
Using openapi-fetch, we create route functions to interact with backend APIs. This library simplifies making API requests by automatically handling the request and response types based on the OpenAPI schema. It ensures type safety and minimizes boilerplate code for API calls and also ensures that the request and response types match those defined in the OpenAPI schema, reducing runtime errors and improving developer experience.

In order to user it, we first instantiate a client using `openapi-fetch` with the base URL of the backend. Then we implement the functions to send HTTP requests to the backend. These functions use the client to send requests and handle responses.

```ts
import createClient, { type Middleware } from "openapi-fetch";
import type { paths } from "../Schema/schema.ts";
import { IPet } from "../types/petTypes.tsx";

// Create client with the base URL of the backend
const client = createClient<paths>({ baseUrl: "http://localhost:3020" });

// Function to get a pet
export const getAPet = async (): Promise<IPet> => {
    try {
        const { data, error } = await client.GET("/petsapp/pet/single", {
            params: {
                query: { name: "ali" }
            }
        });

        if (error) {
            console.error("Error fetching pet:", error);
            throw new Error("Cannot Get");
        }

        return data as IPet;
    } catch (error) {
        console.error("Cannot Get");
        return {} as IPet;
    }
};
```

### Implementing Interceptors
Interceptors allow us to modify requests and responses globally. They are used to add headers, handle authentication, or process responses consistently across different API requests. In order to create interceptors, we use the custom middleware of openapi-fetch along with the openapi-fetch client to apply these modifications globally.

```ts
let accessToken: string | null = null;

// Function to get the access token
const getAccessToken = async (): Promise<{ accessToken: string | null }> => {
    return { accessToken: 'abc-xyz' };
};

// Define interceptor middleware
const myInterceptor: Middleware = {
    async onRequest({ request, schemaPath }) {
        if (schemaPath === "/petsapp/pet/single") {
            return;
        }

        if (!accessToken) {
            const authRes = await getAccessToken();
            if (authRes.accessToken) {
                accessToken = authRes.accessToken;
            } else {
                throw new Error("Authentication failed");
            }
        }

        request.headers.set("Authorization", accessToken);
        return request;
    },

    async onResponse({ response }) {
        const { body, ...resOptions } = response;
        return new Response(body, { ...resOptions, status: 200 });
    },
};

// Attach middleware to the client
client.use(myInterceptor);
```

### Fetching and Managing the Schema
Fetching the schema from Google Drive involves retrieving the latest OpenAPI schema file, comparing it with the local version, and updating the local schema if necessary. This process ensures that the frontend is always using the most current schema, maintaining type safety across the application. Additionally, the method is designed to prevent redundant fetching of the schema file, which saves on network resources and processing time.

1. **Fetch Schema from Google Drive**: To fetch the schema, we use the Google Drive API to list files and download the relevant schema file. This step involves authenticating with Google Drive, listing files in the desired folder, and retrieving the latest schema file based on its name.
2. **Compare Local and Remote Hashes**: To prevent redundant fetching, we compare the hash of the local file that is stored in the .env.local file with the hash embedded in the file name of the remote schema. This ensures that the file is only downloaded if it has changed.
    ```ts
    const localHash = process.env['HASH_STRING'];

    const latestFileName = latestFile.name;
    const remoteHash = latestFileName.split('-').pop()?.replace('.json', '');

    if (localHash === remoteHash) {
        console.log(`Local file hash (${localHash}) is already the latest. Skipping download.`);
        return;
    }
    ```
3.  **Update Local File**: If the local file is outdated, download the new schema and update the local hash value in the .env.local file to reflect the new file hash.
    ```ts
    function updateEnvFile(newHash: string | undefined) {
    if (!newHash) {
        console.error('No new hash to update.');
        return;
    }

    const envPath = '.env.local';
    let envContent = fs.readFileSync(envPath, 'utf-8');
    const hashLinePattern = /^HASH_STRING = .*/m;
    const newHashLine = `HASH_STRING = ${newHash}`;

    if (hashLinePattern.test(envContent)) {
        envContent = envContent.replace(hashLinePattern, newHashLine);
    } else {
        envContent += `\n${newHashLine}`;
    }

    fs.writeFileSync(envPath, envContent, 'utf-8');
    console.log('Updated .env.local with new hash number:', newHash);
    }
    ```
4. **Convert JSON Schema to TypeScript Interfaces**: Once the latest schema has been downloaded, the final step is to convert the JSON schema into TypeScript interfaces. This is done using `openapi-typescript`, which generates TypeScript definitions from the OpenAPI schema. This ensures that the frontend has up-to-date types that match the backend API.
5. **Abstracting the Fetch Process**: To make the schema fetching process more flexible and maintainable, an abstract approach is applied on the frontend. This allows for easy integration with different storage services without modifying the core fetching logic.

    Define an abstract class for file fetching:

    ```ts
    import { StorageService } from './storageService.ts';

    export abstract class FileFetchProcess {
      protected storageService: StorageService;

      constructor(storageService: StorageService) {
        this.storageService = storageService;
      }

      abstract fetchFile(): Promise<void>;
    }
    ```

    Implement the `FileFetchProcess` interface for schema fetching:

    ```ts
    import { FileFetchProcess } from './fetchProcess.ts';
    import { StorageService } from '../services/storageService.ts';
    import dotenv from 'dotenv';
    import fs from 'fs';

    dotenv.config({ path: '.env.local' });

    export class SchemaFetch extends FileFetchProcess {
      constructor(storageService: StorageService) {
        super(storageService);
      }

      private updateEnvFile(newHash: string | undefined) {
        if (!newHash) {
          console.error('No new hash to update.');
          return;
        }

        const envPath = '.env.local';
        let envContent = fs.readFileSync(envPath, 'utf-8');
        const hashLinePattern = /^HASH_STRING = .*/m;
        const newHashLine = `HASH_STRING = ${newHash}`;

        if (hashLinePattern.test(envContent)) {
          envContent = envContent.replace(hashLinePattern, newHashLine);
        } else {
          envContent += `\n${newHashLine}`;
        }

        fs.writeFileSync(envPath, envContent, 'utf-8');
        console.log('Updated .env.local with new hash number:', newHash);
      }

      async fetchFile(): Promise<void> {
        const branchName = process.env['BRANCH_NAME'];
        const localHash = process.env['HASH_STRING'];
        const sourceFilePrefix = `schema-${branchName}`;
        const destinationFilePath = 'src/Schema/schema.json';

        try {
          const files = await this.storageService.listFiles(`'${process.env['DRIVE_FOLDER_ID']}' in parents and trashed=false`);

          if (files.length > 0) {
            let latestFile: { id: string, name: string } | null = null;

            for (const file of files) {
              if (file.name && file.name.startsWith(sourceFilePrefix)) {
                latestFile = file;
                break;
              }
            }

            if (latestFile && latestFile.id) {
              const latestFileName = latestFile.name;
              const remoteHash = latestFileName.split('-').pop()?.replace('.json', '');

              if (localHash === remoteHash) {
                console.log(`Local file hash (${localHash}) is already the latest. Skipping download.`);
                return;
              }

              await this.storageService.getFile(latestFile.id, destinationFilePath);
              console.log('File downloaded successfully.');
              this.updateEnvFile(remoteHash);
            } else {
              console.log(`No file with the prefix ${sourceFilePrefix} found.`);
            }
          } else {
            console.log('No files found in the specified folder.');
          }
        } catch (error) {
          console.error('Error fetching files:', error);
        }
      }
    }
    ```

    Define the abstract class `StorageService` and implement it for Google Drive:

    ```ts
    export interface StorageService {
      listFiles(query: string): Promise<{ id: string, name: string }[]>;
      getFile(fileId: string, destinationPath: string): Promise<void>;
    }

    import { google } from 'googleapis';
    import fs from 'fs';
    import { StorageService } from './storageService.ts';

    export class GoogleDriveService implements StorageService {
      private drive;

      constructor() {
        const auth = new google.auth.GoogleAuth({
          keyFile: './credentials.json',
          scopes: ['https://www.googleapis.com/auth/drive.readonly'],
        });
        this.drive = google.drive({ version: 'v3', auth });
      }

      async listFiles(query: string): Promise<{ id: string, name: string }[]> {
        const listResponse = await this.drive.files.list({
          q: query,
          fields: 'files(id, name)',
        });

        return listResponse.data.files?.map(file => ({
          id: file.id!,
          name: file.name!,
        })) || [];
      }

      async getFile(fileId: string, destinationPath: string): Promise<void> {
        const dest = fs.createWriteStream(destinationPath);

        try {
          const response = await this.drive.files.get(
            { fileId, alt: 'media' },
            { responseType: 'stream' }
          );

          if (response.data && typeof response.data.pipe === 'function') {
            await this.streamToFile(response.data, dest);
          } else {
            throw new Error('Unexpected response format.');
          }
        } catch (err) {
          console.error('Error downloading file:', err);
          throw err;
        }
      }

      private async streamToFile(readableStream: NodeJS.ReadableStream, writableStream: NodeJS.WritableStream): Promise<void> {
        return new Promise<void>((resolve, reject) => {
          readableStream
            .pipe(writableStream)
            .on('finish', () => resolve())
            .on('error', (err) => {
              writableStream.end();
              reject(err);
            });
        });
      }
    }
    ```

By implementing these abstractions, both frontend and backend processes are decoupled from specific storage implementations, ensuring that any changes in storage services require minimal adjustments. This approach enhances maintainability and flexibility across different environments and services.

### Automating Schema Updates with Husky
We use Husky to automate the process of fetching the schema whenever code is committed or built. Husky is a tool that allows us to easily manage Git hooks. By adding a script to run the schema fetching process in the pre-commit or pre-push hook, we ensure that the schema in the frontend is always up-to-date with the backend.

1. Install Husky:
    ```bash
    npm install husky --save-dev
    ```
2. Enable Husky Hooks:

    ```bash
    npx husky install
    ```
3. Create Pre-Commit Hook:

    ```bash
    npx husky add .husky/pre-commit "npm run fetch-swagger"
    ```
4. Update package.json:

    ```json
    "scripts": {
        "prepare": "husky install",
        "fetch-swagger": "tsx src/scripts/fetchingSchema.ts && npx openapi-typescript src/Schema/schema.json -o src/Schema/schema.ts",
    }
    ```

By following these steps, the client-side application maintains type safety and remains synchronized with the backend API changes.
