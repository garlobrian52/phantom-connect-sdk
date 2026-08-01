# MCP Server for Phantom Connect SDK Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create a globally-installable MCP server that enables AI assistants to interact with Phantom wallets via OAuth/DCR authentication.

**Architecture:** Node.js CLI tool using MCP SDK stdio transport. OAuth flow with Dynamic Client Registration (DCR) against auth.phantom.app, local HTTP callback server for authorization code exchange, filesystem session persistence, and PhantomClient integration for wallet operations.

**Tech Stack:** @modelcontextprotocol/sdk, openid-client (OAuth/DCR), PhantomClient, ApiKeyStamper, open (browser launcher), axios, Node.js crypto

---

## Task 1: Package Structure Setup

**Files:**

- Create: `packages/mcp-server/package.json`
- Create: `packages/mcp-server/tsconfig.json`
- Create: `packages/mcp-server/tsup.config.ts`
- Create: `packages/mcp-server/bin/phantom-mcp`
- Create: `packages/mcp-server/src/index.ts`
- Create: `packages/mcp-server/.gitignore`

**Step 1: Create package.json**

Create the package definition with bin entry point and dependencies:

```json
{
  "name": "@phantom/mcp-server",
  "version": "0.1.0",
  "description": "MCP server for Phantom wallet operations",
  "repository": {
    "type": "git",
    "url": "https://github.com/phantom/phantom-connect-sdk",
    "directory": "packages/mcp-server"
  },
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "bin": {
    "phantom-mcp": "./bin/phantom-mcp"
  },
  "scripts": {
    "?pack-release": "When https://github.com/changesets/changesets/issues/432 has a solution we can remove this trick",
    "pack-release": "rimraf ./_release && yarn pack && mkdir ./_release && tar zxvf ./package.tgz --directory ./_release && rm ./package.tgz",
    "build": "rimraf ./dist && tsup",
    "dev": "tsup --watch",
    "clean": "rm -rf dist",
    "test": "jest",
    "test:watch": "jest --watch",
    "lint": "eslint src --ext .ts",
    "check-types": "tsc --noEmit",
    "prettier": "prettier --write \"src/**/*.ts\""
  },
  "devDependencies": {
    "@types/jest": "^29.5.12",
    "@types/node": "^20.11.0",
    "eslint": "8.53.0",
    "jest": "^29.7.0",
    "prettier": "^3.5.2",
    "rimraf": "^6.0.1",
    "ts-jest": "^29.1.2",
    "tsup": "^6.7.0",
    "typescript": "^5.0.4"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.4",
    "@phantom/api-key-stamper": "workspace:^",
    "@phantom/client": "workspace:^",
    "@phantom/constants": "workspace:^",
    "@phantom/crypto": "workspace:^",
    "axios": "^1.10.0",
    "openid-client": "^6.1.3",
    "open": "^10.1.0"
  },
  "files": ["dist", "bin"],
  "publishConfig": {
    "directory": "_release/package"
  }
}
```

**Step 2: Create tsconfig.json**

Configure TypeScript for Node.js environment:

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "module": "commonjs",
    "target": "es2020",
    "lib": ["es2020"],
    "moduleResolution": "node",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "strict": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

**Step 3: Create tsup.config.ts**

Configure build to output CJS format for Node.js:

```typescript
import { defineConfig } from "tsup";

export default defineConfig({
  entry: ["src/index.ts"],
  format: ["cjs"],
  dts: true,
  clean: true,
  platform: "node",
  target: "node18",
  shims: true,
});
```

**Step 4: Create bin/phantom-mcp**

Create executable entry point:

```bash
#!/usr/bin/env node
require('../dist/index.js');
```

**Step 5: Create placeholder src/index.ts**

```typescript
#!/usr/bin/env node

async function main() {
  console.error("[INFO] Phantom MCP Server starting...");
}

main().catch(error => {
  console.error("[ERROR] Fatal error:", error);
  process.exit(1);
});
```

**Step 6: Create .gitignore**

```
dist/
node_modules/
*.tgz
_release/
.turbo/
```

**Step 7: Make bin/phantom-mcp executable**

Run: `chmod +x packages/mcp-server/bin/phantom-mcp`
Expected: File becomes executable

**Step 8: Build and verify**

Run: `cd packages/mcp-server && yarn install && yarn build`
Expected: dist/ directory created with index.js

**Step 9: Commit**

```bash
git add packages/mcp-server
git commit -m "feat(mcp-server): initialize package structure

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 2: Logger Utility (Stderr-only)

**Files:**

- Create: `packages/mcp-server/src/utils/logger.ts`

**Step 1: Create logger implementation**

CRITICAL: All logging must go to stderr since stdout is reserved for JSON-RPC.

```typescript
/**
 * Logger utility for MCP server
 * CRITICAL: All output goes to stderr - stdout is reserved for JSON-RPC messages
 */
export class Logger {
  private context: string;

  constructor(context: string = "MCP") {
    this.context = context;
  }

  private log(level: string, ...args: any[]): void {
    const timestamp = new Date().toISOString();
    const message = `[${timestamp}] [${level}] [${this.context}] ${args.join(" ")}\n`;
    process.stderr.write(message);
  }

  info(...args: any[]): void {
    this.log("INFO", ...args);
  }

  error(...args: any[]): void {
    this.log("ERROR", ...args);
  }

  warn(...args: any[]): void {
    this.log("WARN", ...args);
  }

  debug(...args: any[]): void {
    if (process.env.DEBUG || process.env.PHANTOM_MCP_DEBUG) {
      this.log("DEBUG", ...args);
    }
  }

  /**
   * Create a child logger with additional context
   */
  child(childContext: string): Logger {
    return new Logger(`${this.context}:${childContext}`);
  }
}

// Export singleton for convenience
export const logger = new Logger();
```

**Step 2: Create test file**

Create: `packages/mcp-server/src/utils/logger.test.ts`

```typescript
import { Logger } from "./logger";

describe("Logger", () => {
  let stderrSpy: jest.SpyInstance;

  beforeEach(() => {
    stderrSpy = jest.spyOn(process.stderr, "write").mockImplementation();
  });

  afterEach(() => {
    stderrSpy.mockRestore();
  });

  it("should log to stderr, not stdout", () => {
    const stdoutSpy = jest.spyOn(process.stdout, "write").mockImplementation();
    const logger = new Logger("test");

    logger.info("test message");

    expect(stderrSpy).toHaveBeenCalled();
    expect(stdoutSpy).not.toHaveBeenCalled();

    stdoutSpy.mockRestore();
  });

  it("should include timestamp, level, and context", () => {
    const logger = new Logger("TestContext");
    logger.info("test message");

    const output = stderrSpy.mock.calls[0][0];
    expect(output).toContain("[INFO]");
    expect(output).toContain("[TestContext]");
    expect(output).toContain("test message");
  });

  it("should support child loggers", () => {
    const parent = new Logger("parent");
    const child = parent.child("child");

    child.info("test");

    const output = stderrSpy.mock.calls[0][0];
    expect(output).toContain("[parent:child]");
  });
});
```

**Step 3: Run test to verify it fails**

Run: `cd packages/mcp-server && yarn test`
Expected: Test fails (logger not implemented yet)

**Step 4: Implement logger (already done in Step 1)**

**Step 5: Run test to verify it passes**

Run: `cd packages/mcp-server && yarn test`
Expected: All logger tests pass

**Step 6: Commit**

```bash
git add packages/mcp-server/src/utils/logger.ts packages/mcp-server/src/utils/logger.test.ts
git commit -m "feat(mcp-server): add stderr-only logger utility

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 3: Session Types and Storage

**Files:**

- Create: `packages/mcp-server/src/session/types.ts`
- Create: `packages/mcp-server/src/session/storage.ts`
- Create: `packages/mcp-server/src/session/storage.test.ts`

**Step 1: Define session types**

```typescript
export interface SessionData {
  walletId: string;
  organizationId: string;
  authUserId: string;
  accessToken: string;
  refreshToken: string;
  expiresAt: number; // Unix timestamp in seconds
  stamperKeys: {
    publicKey: string; // bs58 format
    secretKey: string; // bs58 format
  };
  createdAt: number; // Unix timestamp in seconds
  updatedAt: number; // Unix timestamp in seconds
}

export interface OAuthCallbackParams {
  code: string;
  state: string;
  wallet_id: string;
  organization_id: string;
  auth_user_id: string;
}

export interface OAuthTokens {
  access_token: string;
  refresh_token: string;
  expires_in: number; // Seconds until expiration
}

export interface DCRClientConfig {
  client_id: string;
  client_secret: string;
  client_id_issued_at: number;
}
```

**Step 2: Create storage implementation**

```typescript
import fs from "fs";
import path from "path";
import os from "os";
import { Logger } from "../utils/logger";
import type { SessionData } from "./types";

const logger = new Logger("Storage");

export class SessionStorage {
  private sessionDir: string;
  private sessionFile: string;

  constructor(sessionDir?: string) {
    this.sessionDir = sessionDir || path.join(os.homedir(), ".phantom-mcp");
    this.sessionFile = path.join(this.sessionDir, "session.json");
  }

  /**
   * Ensure session directory exists with secure permissions
   */
  private ensureSessionDir(): void {
    if (!fs.existsSync(this.sessionDir)) {
      logger.debug("Creating session directory:", this.sessionDir);
      fs.mkdirSync(this.sessionDir, { recursive: true, mode: 0o700 });
    } else {
      // Verify permissions on existing directory
      fs.chmodSync(this.sessionDir, 0o700);
    }
  }

  /**
   * Load session from disk
   * Returns null if session doesn't exist
   */
  load(): SessionData | null {
    try {
      if (!fs.existsSync(this.sessionFile)) {
        logger.debug("No session file found");
        return null;
      }

      const data = fs.readFileSync(this.sessionFile, "utf-8");
      const session = JSON.parse(data) as SessionData;

      logger.debug("Session loaded successfully");
      return session;
    } catch (error) {
      logger.error("Failed to load session:", error);
      return null;
    }
  }

  /**
   * Save session to disk with secure permissions
   */
  save(session: SessionData): void {
    try {
      this.ensureSessionDir();

      const data = JSON.stringify(session, null, 2);

      // Write to temp file first
      const tempFile = `${this.sessionFile}.tmp`;
      fs.writeFileSync(tempFile, data, { mode: 0o600 });

      // Atomic rename
      fs.renameSync(tempFile, this.sessionFile);

      // Ensure permissions are correct
      fs.chmodSync(this.sessionFile, 0o600);

      logger.info("Session saved successfully");
    } catch (error) {
      logger.error("Failed to save session:", error);
      throw error;
    }
  }

  /**
   * Delete session file
   */
  delete(): void {
    try {
      if (fs.existsSync(this.sessionFile)) {
        fs.unlinkSync(this.sessionFile);
        logger.info("Session deleted");
      }
    } catch (error) {
      logger.error("Failed to delete session:", error);
      throw error;
    }
  }

  /**
   * Check if session is expired
   * Uses 5-minute buffer to refresh before actual expiration
   */
  isExpired(session: SessionData): boolean {
    const bufferSeconds = 5 * 60; // 5 minutes
    const now = Math.floor(Date.now() / 1000);
    return session.expiresAt - bufferSeconds <= now;
  }
}
```

**Step 3: Create test file**

```typescript
import fs from "fs";
import path from "path";
import os from "os";
import { SessionStorage } from "./storage";
import type { SessionData } from "./types";

describe("SessionStorage", () => {
  let tempDir: string;
  let storage: SessionStorage;
  let mockSession: SessionData;

  beforeEach(() => {
    // Create temp directory for testing
    tempDir = path.join(os.tmpdir(), `phantom-mcp-test-${Date.now()}`);
    storage = new SessionStorage(tempDir);

    mockSession = {
      walletId: "wallet-123",
      organizationId: "org-456",
      authUserId: "user-789",
      accessToken: "access-token",
      refreshToken: "refresh-token",
      expiresAt: Math.floor(Date.now() / 1000) + 3600,
      stamperKeys: {
        publicKey: "public-key",
        secretKey: "secret-key",
      },
      createdAt: Math.floor(Date.now() / 1000),
      updatedAt: Math.floor(Date.now() / 1000),
    };
  });

  afterEach(() => {
    // Clean up temp directory
    if (fs.existsSync(tempDir)) {
      fs.rmSync(tempDir, { recursive: true, force: true });
    }
  });

  it("should return null when no session exists", () => {
    const session = storage.load();
    expect(session).toBeNull();
  });

  it("should save and load session", () => {
    storage.save(mockSession);
    const loaded = storage.load();

    expect(loaded).toEqual(mockSession);
  });

  it("should create directory with secure permissions", () => {
    storage.save(mockSession);

    const stats = fs.statSync(tempDir);
    // Check that only owner has rwx permissions (0o700)
    expect(stats.mode & 0o777).toBe(0o700);
  });

  it("should save file with secure permissions", () => {
    storage.save(mockSession);

    const sessionFile = path.join(tempDir, "session.json");
    const stats = fs.statSync(sessionFile);
    // Check that only owner has rw permissions (0o600)
    expect(stats.mode & 0o777).toBe(0o600);
  });

  it("should delete session", () => {
    storage.save(mockSession);
    expect(storage.load()).not.toBeNull();

    storage.delete();
    expect(storage.load()).toBeNull();
  });

  it("should detect expired sessions", () => {
    const expiredSession = {
      ...mockSession,
      expiresAt: Math.floor(Date.now() / 1000) - 3600, // 1 hour ago
    };

    expect(storage.isExpired(expiredSession)).toBe(true);
    expect(storage.isExpired(mockSession)).toBe(false);
  });

  it("should use 5-minute buffer for expiration", () => {
    const almostExpiredSession = {
      ...mockSession,
      expiresAt: Math.floor(Date.now() / 1000) + 4 * 60, // 4 minutes from now
    };

    // Should be considered expired due to 5-minute buffer
    expect(storage.isExpired(almostExpiredSession)).toBe(true);
  });
});
```

**Step 4: Run tests**

Run: `cd packages/mcp-server && yarn test`
Expected: All storage tests pass

**Step 5: Commit**

```bash
git add packages/mcp-server/src/session/
git commit -m "feat(mcp-server): add session storage with secure file permissions

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 4: Dynamic Client Registration (DCR)

**Files:**

- Create: `packages/mcp-server/src/auth/dcr.ts`
- Create: `packages/mcp-server/src/auth/dcr.test.ts`

**Step 1: Implement DCR client**

```typescript
import axios from "axios";
import { Logger } from "../utils/logger";
import type { DCRClientConfig } from "../session/types";

const logger = new Logger("DCR");

export interface DCROptions {
  authBaseUrl?: string;
  appId?: string;
}

export class DCRClient {
  private authBaseUrl: string;
  private appId: string;

  constructor(options: DCROptions = {}) {
    this.authBaseUrl = options.authBaseUrl || "https://auth.phantom.app";
    this.appId = options.appId || "phantom-mcp";
  }

  /**
   * Register OAuth client dynamically per RFC 7591
   * @param redirectUri - Callback URL for OAuth flow
   */
  async register(redirectUri: string): Promise<DCRClientConfig> {
    const registrationUrl = `${this.authBaseUrl}/oauth/register`;

    const payload = {
      client_name: `${this.appId}-${Date.now()}`,
      redirect_uris: [redirectUri],
      grant_types: ["authorization_code", "refresh_token"],
      response_types: ["code"],
      application_type: "native",
      token_endpoint_auth_method: "client_secret_basic",
    };

    logger.debug("Registering OAuth client...");
    logger.debug("Registration URL:", registrationUrl);
    logger.debug("Redirect URI:", redirectUri);

    try {
      const response = await axios.post(registrationUrl, payload, {
        headers: {
          "Content-Type": "application/json",
        },
      });

      const config: DCRClientConfig = {
        client_id: response.data.client_id,
        client_secret: response.data.client_secret,
        client_id_issued_at: response.data.client_id_issued_at,
      };

      logger.info("OAuth client registered successfully");
      logger.debug("Client ID:", config.client_id);

      return config;
    } catch (error: any) {
      logger.error("DCR registration failed:", error.response?.data || error.message);
      throw new Error(`DCR registration failed: ${error.response?.data?.error || error.message}`);
    }
  }
}
```

**Step 2: Create test file**

```typescript
import axios from "axios";
import { DCRClient } from "./dcr";

jest.mock("axios");
const mockedAxios = axios as jest.Mocked<typeof axios>;

describe("DCRClient", () => {
  let dcrClient: DCRClient;

  beforeEach(() => {
    jest.clearAllMocks();
    dcrClient = new DCRClient({ appId: "test-app" });
  });

  it("should register client with correct payload", async () => {
    const mockResponse = {
      data: {
        client_id: "test-client-id",
        client_secret: "test-client-secret",
        client_id_issued_at: 1234567890,
      },
    };

    mockedAxios.post.mockResolvedValue(mockResponse);

    const redirectUri = "http://localhost:8080/callback";
    const result = await dcrClient.register(redirectUri);

    expect(mockedAxios.post).toHaveBeenCalledWith(
      "https://auth.phantom.app/oauth/register",
      expect.objectContaining({
        redirect_uris: [redirectUri],
        grant_types: ["authorization_code", "refresh_token"],
        response_types: ["code"],
        application_type: "native",
      }),
      expect.any(Object),
    );

    expect(result).toEqual({
      client_id: "test-client-id",
      client_secret: "test-client-secret",
      client_id_issued_at: 1234567890,
    });
  });

  it("should handle registration errors", async () => {
    mockedAxios.post.mockRejectedValue({
      response: {
        data: { error: "invalid_request" },
      },
    });

    await expect(dcrClient.register("http://localhost:8080/callback")).rejects.toThrow("DCR registration failed");
  });
});
```

**Step 3: Run tests**

Run: `cd packages/mcp-server && yarn test`
Expected: All DCR tests pass

**Step 4: Commit**

```bash
git add packages/mcp-server/src/auth/dcr.ts packages/mcp-server/src/auth/dcr.test.ts
git commit -m "feat(mcp-server): add dynamic client registration (DCR)

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 5: OAuth Callback Server

**Files:**

- Create: `packages/mcp-server/src/auth/callback-server.ts`
- Create: `packages/mcp-server/src/auth/callback-server.test.ts`

**Step 1: Implement callback server**

```typescript
import http from "http";
import { URL } from "url";
import { Logger } from "../utils/logger";
import type { OAuthCallbackParams } from "../session/types";

const logger = new Logger("CallbackServer");

export interface CallbackServerOptions {
  port?: number;
  host?: string;
  timeoutMs?: number;
}

export class CallbackServer {
  private server: http.Server | null = null;
  private port: number;
  private host: string;
  private timeoutMs: number;

  constructor(options: CallbackServerOptions = {}) {
    this.port = options.port || 8080;
    this.host = options.host || "localhost";
    this.timeoutMs = options.timeoutMs || 5 * 60 * 1000; // 5 minutes
  }

  /**
   * Start HTTP server and wait for OAuth callback
   * Returns the callback parameters or throws on timeout/error
   */
  async waitForCallback(expectedState: string): Promise<OAuthCallbackParams> {
    return new Promise((resolve, reject) => {
      let timeoutHandle: NodeJS.Timeout;

      this.server = http.createServer((req, res) => {
        try {
          const url = new URL(req.url || "", `http://${this.host}:${this.port}`);

          if (url.pathname !== "/callback") {
            res.writeHead(404);
            res.end("Not Found");
            return;
          }

          // Extract OAuth parameters
          const code = url.searchParams.get("code");
          const state = url.searchParams.get("state");
          const walletId = url.searchParams.get("wallet_id");
          const organizationId = url.searchParams.get("organization_id");
          const authUserId = url.searchParams.get("auth_user_id");
          const error = url.searchParams.get("error");
          const errorDescription = url.searchParams.get("error_description");

          // Check for OAuth errors
          if (error) {
            logger.error("OAuth error:", error, errorDescription);
            this.sendErrorPage(res, error, errorDescription || "Unknown error");
            clearTimeout(timeoutHandle);
            this.close();
            reject(new Error(`OAuth error: ${error} - ${errorDescription}`));
            return;
          }

          // Validate required parameters
          if (!code || !state || !walletId || !organizationId || !authUserId) {
            logger.error("Missing required OAuth parameters");
            this.sendErrorPage(res, "invalid_request", "Missing required parameters");
            clearTimeout(timeoutHandle);
            this.close();
            reject(new Error("Missing required OAuth parameters"));
            return;
          }

          // Validate state matches
          if (state !== expectedState) {
            logger.error("State mismatch:", state, "expected:", expectedState);
            this.sendErrorPage(res, "invalid_state", "State parameter mismatch");
            clearTimeout(timeoutHandle);
            this.close();
            reject(new Error("OAuth state mismatch - possible CSRF attack"));
            return;
          }

          // Success - send success page
          this.sendSuccessPage(res);

          clearTimeout(timeoutHandle);

          // Close server after short delay to allow response to send
          setTimeout(() => {
            this.close();
            resolve({
              code,
              state,
              wallet_id: walletId,
              organization_id: organizationId,
              auth_user_id: authUserId,
            });
          }, 100);
        } catch (error) {
          logger.error("Error handling callback:", error);
          res.writeHead(500);
          res.end("Internal Server Error");
          clearTimeout(timeoutHandle);
          this.close();
          reject(error);
        }
      });

      // Set up timeout
      timeoutHandle = setTimeout(() => {
        logger.error("Callback timeout after", this.timeoutMs, "ms");
        this.close();
        reject(new Error("OAuth callback timeout"));
      }, this.timeoutMs);

      // Start server
      this.server.listen(this.port, this.host, () => {
        logger.info(`Callback server listening on http://${this.host}:${this.port}/callback`);
      });

      // Handle server errors
      this.server.on("error", (error: any) => {
        logger.error("Server error:", error);
        clearTimeout(timeoutHandle);
        reject(error);
      });
    });
  }

  /**
   * Close the HTTP server
   */
  private close(): void {
    if (this.server) {
      this.server.close();
      this.server = null;
      logger.debug("Callback server closed");
    }
  }

  /**
   * Get the callback URL for this server
   */
  getCallbackUrl(): string {
    return `http://${this.host}:${this.port}/callback`;
  }

  /**
   * Send success HTML page
   */
  private sendSuccessPage(res: http.ServerResponse): void {
    const html = `
<!DOCTYPE html>
<html>
<head>
  <title>Phantom MCP - Authorization Successful</title>
  <style>
    body { font-family: sans-serif; text-align: center; padding: 50px; background: #f5f5f5; }
    .container { max-width: 500px; margin: 0 auto; background: white; padding: 40px; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
    h1 { color: #4CAF50; }
    p { color: #666; line-height: 1.6; }
  </style>
</head>
<body>
  <div class="container">
    <h1>✓ Authorization Successful</h1>
    <p>Your Phantom wallet has been connected successfully.</p>
    <p>You can now close this window and return to your terminal.</p>
  </div>
</body>
</html>
    `;
    res.writeHead(200, { "Content-Type": "text/html" });
    res.end(html);
  }

  /**
   * Send error HTML page
   */
  private sendErrorPage(res: http.ServerResponse, error: string, description: string): void {
    const html = `
<!DOCTYPE html>
<html>
<head>
  <title>Phantom MCP - Authorization Failed</title>
  <style>
    body { font-family: sans-serif; text-align: center; padding: 50px; background: #f5f5f5; }
    .container { max-width: 500px; margin: 0 auto; background: white; padding: 40px; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
    h1 { color: #f44336; }
    p { color: #666; line-height: 1.6; }
    .error-code { font-family: monospace; background: #f5f5f5; padding: 10px; border-radius: 4px; margin: 20px 0; }
  </style>
</head>
<body>
  <div class="container">
    <h1>✗ Authorization Failed</h1>
    <p>${description}</p>
    <div class="error-code">${error}</div>
    <p>Please try again or contact support if the problem persists.</p>
  </div>
</body>
</html>
    `;
    res.writeHead(400, { "Content-Type": "text/html" });
    res.end(html);
  }
}
```

**Step 2: Create test file**

```typescript
import http from "http";
import { CallbackServer } from "./callback-server";

describe("CallbackServer", () => {
  let server: CallbackServer;

  afterEach(() => {
    // Ensure server is closed after each test
    if (server) {
      (server as any).close();
    }
  });

  it("should start server on specified port", async () => {
    server = new CallbackServer({ port: 8081, timeoutMs: 1000 });

    const callbackPromise = server.waitForCallback("test-state");

    // Give server time to start
    await new Promise(resolve => setTimeout(resolve, 100));

    // Verify server is listening
    const response = await fetch(
      "http://localhost:8081/callback?code=test&state=test-state&wallet_id=w1&organization_id=o1&auth_user_id=u1",
    );
    expect(response.status).toBe(200);

    const result = await callbackPromise;
    expect(result.code).toBe("test");
  });

  it("should validate state parameter", async () => {
    server = new CallbackServer({ port: 8082, timeoutMs: 1000 });

    const callbackPromise = server.waitForCallback("expected-state");

    await new Promise(resolve => setTimeout(resolve, 100));

    // Send callback with wrong state
    await fetch(
      "http://localhost:8082/callback?code=test&state=wrong-state&wallet_id=w1&organization_id=o1&auth_user_id=u1",
    );

    await expect(callbackPromise).rejects.toThrow("state mismatch");
  });

  it("should timeout if no callback received", async () => {
    server = new CallbackServer({ port: 8083, timeoutMs: 100 });

    const callbackPromise = server.waitForCallback("test-state");

    await expect(callbackPromise).rejects.toThrow("timeout");
  });

  it("should return callback URL", () => {
    server = new CallbackServer({ port: 8084, host: "localhost" });
    expect(server.getCallbackUrl()).toBe("http://localhost:8084/callback");
  });
});
```

**Step 3: Run tests**

Run: `cd packages/mcp-server && yarn test`
Expected: All callback server tests pass

**Step 4: Commit**

```bash
git add packages/mcp-server/src/auth/callback-server.ts packages/mcp-server/src/auth/callback-server.test.ts
git commit -m "feat(mcp-server): add OAuth callback HTTP server

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 6: OAuth Flow Implementation

**Files:**

- Create: `packages/mcp-server/src/auth/oauth.ts`
- Create: `packages/mcp-server/src/auth/oauth.test.ts`

**Step 1: Implement OAuth flow with PKCE**

```typescript
import crypto from "crypto";
import axios from "axios";
import open from "open";
import { Logger } from "../utils/logger";
import { DCRClient } from "./dcr";
import { CallbackServer } from "./callback-server";
import type { OAuthTokens, DCRClientConfig, OAuthCallbackParams } from "../session/types";
import { base64urlEncode } from "@phantom/crypto";

const logger = new Logger("OAuth");

export interface OAuthFlowOptions {
  authBaseUrl?: string;
  connectBaseUrl?: string;
  callbackPort?: number;
  appId?: string;
}

export interface OAuthFlowResult {
  tokens: OAuthTokens;
  walletId: string;
  organizationId: string;
  authUserId: string;
}

export class OAuthFlow {
  private authBaseUrl: string;
  private connectBaseUrl: string;
  private callbackPort: number;
  private appId: string;
  private dcrClient: DCRClient;

  constructor(options: OAuthFlowOptions = {}) {
    this.authBaseUrl = options.authBaseUrl || "https://auth.phantom.app";
    this.connectBaseUrl = options.connectBaseUrl || "https://connect.phantom.app";
    this.callbackPort = options.callbackPort || 8080;
    this.appId = options.appId || "phantom-mcp";
    this.dcrClient = new DCRClient({ authBaseUrl: this.authBaseUrl, appId: this.appId });
  }

  /**
   * Generate PKCE code verifier and challenge
   */
  private generatePKCE(): { codeVerifier: string; codeChallenge: string } {
    // Generate random code verifier (43-128 characters)
    const codeVerifier = base64urlEncode(crypto.randomBytes(32));

    // Generate code challenge using S256 method
    const hash = crypto.createHash("sha256").update(codeVerifier).digest();
    const codeChallenge = base64urlEncode(hash);

    return { codeVerifier, codeChallenge };
  }

  /**
   * Generate random state parameter for CSRF protection
   */
  private generateState(): string {
    return base64urlEncode(crypto.randomBytes(32));
  }

  /**
   * Build authorization URL
   */
  private buildAuthorizationUrl(clientId: string, redirectUri: string, state: string, codeChallenge: string): string {
    const params = new URLSearchParams({
      client_id: clientId,
      redirect_uri: redirectUri,
      response_type: "code",
      state: state,
      code_challenge: codeChallenge,
      code_challenge_method: "S256",
      scope: "openid wallet:read wallet:write",
    });

    return `${this.connectBaseUrl}/authorize?${params.toString()}`;
  }

  /**
   * Exchange authorization code for tokens
   */
  private async exchangeCode(
    code: string,
    codeVerifier: string,
    redirectUri: string,
    clientConfig: DCRClientConfig,
  ): Promise<OAuthTokens> {
    const tokenUrl = `${this.authBaseUrl}/oauth/token`;

    const params = new URLSearchParams({
      grant_type: "authorization_code",
      code: code,
      redirect_uri: redirectUri,
      code_verifier: codeVerifier,
      client_id: clientConfig.client_id,
      client_secret: clientConfig.client_secret,
    });

    logger.debug("Exchanging authorization code for tokens...");

    try {
      const response = await axios.post(tokenUrl, params, {
        headers: {
          "Content-Type": "application/x-www-form-urlencoded",
        },
      });

      const tokens: OAuthTokens = {
        access_token: response.data.access_token,
        refresh_token: response.data.refresh_token,
        expires_in: response.data.expires_in,
      };

      logger.info("Successfully exchanged code for tokens");
      return tokens;
    } catch (error: any) {
      logger.error("Token exchange failed:", error.response?.data || error.message);
      throw new Error(`Token exchange failed: ${error.response?.data?.error || error.message}`);
    }
  }

  /**
   * Execute full OAuth flow
   */
  async authenticate(): Promise<OAuthFlowResult> {
    logger.info("Starting OAuth flow...");

    // Step 1: Register OAuth client via DCR
    const callbackServer = new CallbackServer({ port: this.callbackPort });
    const redirectUri = callbackServer.getCallbackUrl();

    logger.info("Registering OAuth client...");
    const clientConfig = await this.dcrClient.register(redirectUri);

    // Step 2: Generate PKCE parameters
    const { codeVerifier, codeChallenge } = this.generatePKCE();
    const state = this.generateState();

    // Step 3: Build authorization URL and open browser
    const authUrl = this.buildAuthorizationUrl(clientConfig.client_id, redirectUri, state, codeChallenge);

    logger.info("Opening browser for authorization...");
    logger.info("Authorization URL:", authUrl);

    try {
      await open(authUrl);
    } catch (error) {
      logger.warn("Failed to open browser automatically:", error);
      logger.info("Please open this URL in your browser:", authUrl);
    }

    // Step 4: Wait for callback
    logger.info("Waiting for OAuth callback...");
    const callbackParams: OAuthCallbackParams = await callbackServer.waitForCallback(state);

    logger.info("Received OAuth callback");
    logger.debug("Wallet ID:", callbackParams.wallet_id);
    logger.debug("Organization ID:", callbackParams.organization_id);

    // Step 5: Exchange authorization code for tokens
    const tokens = await this.exchangeCode(callbackParams.code, codeVerifier, redirectUri, clientConfig);

    logger.info("OAuth flow completed successfully");

    return {
      tokens,
      walletId: callbackParams.wallet_id,
      organizationId: callbackParams.organization_id,
      authUserId: callbackParams.auth_user_id,
    };
  }

  /**
   * Refresh access token using refresh token
   */
  async refreshToken(refreshToken: string, clientConfig: DCRClientConfig): Promise<OAuthTokens> {
    const tokenUrl = `${this.authBaseUrl}/oauth/token`;

    const params = new URLSearchParams({
      grant_type: "refresh_token",
      refresh_token: refreshToken,
      client_id: clientConfig.client_id,
      client_secret: clientConfig.client_secret,
    });

    logger.debug("Refreshing access token...");

    try {
      const response = await axios.post(tokenUrl, params, {
        headers: {
          "Content-Type": "application/x-www-form-urlencoded",
        },
      });

      const tokens: OAuthTokens = {
        access_token: response.data.access_token,
        refresh_token: response.data.refresh_token || refreshToken, // Some servers don't return new refresh token
        expires_in: response.data.expires_in,
      };

      logger.info("Successfully refreshed access token");
      return tokens;
    } catch (error: any) {
      logger.error("Token refresh failed:", error.response?.data || error.message);
      throw new Error(`Token refresh failed: ${error.response?.data?.error || error.message}`);
    }
  }
}
```

**Step 2: Create test file**

```typescript
import { OAuthFlow } from "./oauth";
import { DCRClient } from "./dcr";
import { CallbackServer } from "./callback-server";
import open from "open";

jest.mock("./dcr");
jest.mock("./callback-server");
jest.mock("open");

describe("OAuthFlow", () => {
  let oauthFlow: OAuthFlow;

  beforeEach(() => {
    jest.clearAllMocks();
    oauthFlow = new OAuthFlow({ callbackPort: 8080 });
  });

  it("should complete full OAuth flow", async () => {
    // Mock DCR registration
    const mockClientConfig = {
      client_id: "test-client-id",
      client_secret: "test-client-secret",
      client_id_issued_at: 1234567890,
    };
    (DCRClient.prototype.register as jest.Mock).mockResolvedValue(mockClientConfig);

    // Mock callback server
    const mockCallbackParams = {
      code: "auth-code",
      state: expect.any(String),
      wallet_id: "wallet-123",
      organization_id: "org-456",
      auth_user_id: "user-789",
    };
    (CallbackServer.prototype.waitForCallback as jest.Mock).mockResolvedValue(mockCallbackParams);
    (CallbackServer.prototype.getCallbackUrl as jest.Mock).mockReturnValue("http://localhost:8080/callback");

    // Mock browser open
    (open as jest.Mock).mockResolvedValue({});

    // Mock token exchange - need to mock axios
    const axios = require("axios");
    axios.post = jest.fn().mockResolvedValue({
      data: {
        access_token: "access-token",
        refresh_token: "refresh-token",
        expires_in: 3600,
      },
    });

    const result = await oauthFlow.authenticate();

    expect(result).toEqual({
      tokens: {
        access_token: "access-token",
        refresh_token: "refresh-token",
        expires_in: 3600,
      },
      walletId: "wallet-123",
      organizationId: "org-456",
      authUserId: "user-789",
    });
  });
});
```

**Step 3: Run tests**

Run: `cd packages/mcp-server && yarn test`
Expected: All OAuth tests pass

**Step 4: Commit**

```bash
git add packages/mcp-server/src/auth/oauth.ts packages/mcp-server/src/auth/oauth.test.ts
git commit -m "feat(mcp-server): add OAuth flow with PKCE

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 7: Session Manager

**Files:**

- Create: `packages/mcp-server/src/session/manager.ts`
- Create: `packages/mcp-server/src/session/manager.test.ts`

**Step 1: Implement session manager**

```typescript
import { PhantomClient } from "@phantom/client";
import { ApiKeyStamper } from "@phantom/api-key-stamper";
import { createKeyPair } from "@phantom/crypto";
import { Logger } from "../utils/logger";
import { SessionStorage } from "./storage";
import { OAuthFlow } from "../auth/oauth";
import type { SessionData } from "./types";

const logger = new Logger("SessionManager");

export interface SessionManagerOptions {
  authBaseUrl?: string;
  connectBaseUrl?: string;
  apiBaseUrl?: string;
  callbackPort?: number;
  appId?: string;
  sessionDir?: string;
}

export class SessionManager {
  private storage: SessionStorage;
  private oauthFlow: OAuthFlow;
  private apiBaseUrl: string;
  private session: SessionData | null = null;
  private client: PhantomClient | null = null;

  constructor(options: SessionManagerOptions = {}) {
    this.storage = new SessionStorage(options.sessionDir);
    this.oauthFlow = new OAuthFlow({
      authBaseUrl: options.authBaseUrl,
      connectBaseUrl: options.connectBaseUrl,
      callbackPort: options.callbackPort,
      appId: options.appId,
    });
    this.apiBaseUrl = options.apiBaseUrl || "https://api.phantom.app";
  }

  /**
   * Initialize session - load from disk or authenticate
   */
  async initialize(): Promise<void> {
    logger.info("Initializing session...");

    // Try to load existing session
    this.session = this.storage.load();

    if (this.session) {
      logger.info("Loaded existing session");

      // Check if session is expired
      if (this.storage.isExpired(this.session)) {
        logger.warn("Session expired, re-authenticating...");
        await this.authenticate();
      } else {
        logger.info("Session is valid");
        this.createClient();
      }
    } else {
      logger.info("No existing session, authenticating...");
      await this.authenticate();
    }
  }

  /**
   * Perform OAuth authentication and create new session
   */
  private async authenticate(): Promise<void> {
    try {
      logger.info("Starting authentication flow...");

      // Execute OAuth flow
      const result = await this.oauthFlow.authenticate();

      // Generate stamper keypair
      const keyPair = createKeyPair();

      // Calculate expiration timestamp
      const now = Math.floor(Date.now() / 1000);
      const expiresAt = now + result.tokens.expires_in;

      // Create session data
      this.session = {
        walletId: result.walletId,
        organizationId: result.organizationId,
        authUserId: result.authUserId,
        accessToken: result.tokens.access_token,
        refreshToken: result.tokens.refresh_token,
        expiresAt: expiresAt,
        stamperKeys: {
          publicKey: keyPair.publicKey,
          secretKey: keyPair.secretKey,
        },
        createdAt: now,
        updatedAt: now,
      };

      // Save session to disk
      this.storage.save(this.session);

      // Create PhantomClient
      this.createClient();

      logger.info("Authentication successful");
    } catch (error) {
      logger.error("Authentication failed:", error);
      // Clean up any partial session
      this.storage.delete();
      throw error;
    }
  }

  /**
   * Create PhantomClient with current session
   */
  private createClient(): void {
    if (!this.session) {
      throw new Error("Cannot create client without session");
    }

    // Create API key stamper with session keypair
    const stamper = new ApiKeyStamper({
      apiSecretKey: this.session.stamperKeys.secretKey,
    });

    // Create PhantomClient
    this.client = new PhantomClient(
      {
        apiBaseUrl: this.apiBaseUrl,
        organizationId: this.session.organizationId,
        walletType: "user-wallet",
      },
      stamper,
    );

    logger.debug("PhantomClient created");
  }

  /**
   * Get PhantomClient instance
   */
  getClient(): PhantomClient {
    if (!this.client) {
      throw new Error("Client not initialized - call initialize() first");
    }
    return this.client;
  }

  /**
   * Get current session data
   */
  getSession(): SessionData {
    if (!this.session) {
      throw new Error("No active session");
    }
    return this.session;
  }

  /**
   * Clear session and re-authenticate
   */
  async resetSession(): Promise<void> {
    logger.info("Resetting session...");
    this.storage.delete();
    this.session = null;
    this.client = null;
    await this.authenticate();
  }
}
```

**Step 2: Create test file**

```typescript
import { SessionManager } from "./manager";
import { SessionStorage } from "./storage";
import { OAuthFlow } from "../auth/oauth";
import type { SessionData } from "./types";

jest.mock("./storage");
jest.mock("../auth/oauth");

describe("SessionManager", () => {
  let manager: SessionManager;
  let mockStorage: jest.Mocked<SessionStorage>;
  let mockOAuthFlow: jest.Mocked<OAuthFlow>;

  beforeEach(() => {
    jest.clearAllMocks();
    mockStorage = new SessionStorage() as jest.Mocked<SessionStorage>;
    mockOAuthFlow = new OAuthFlow() as jest.Mocked<OAuthFlow>;

    manager = new SessionManager();
    (manager as any).storage = mockStorage;
    (manager as any).oauthFlow = mockOAuthFlow;
  });

  it("should load existing valid session", async () => {
    const mockSession: SessionData = {
      walletId: "wallet-123",
      organizationId: "org-456",
      authUserId: "user-789",
      accessToken: "access-token",
      refreshToken: "refresh-token",
      expiresAt: Math.floor(Date.now() / 1000) + 3600,
      stamperKeys: {
        publicKey: "public-key",
        secretKey: "secret-key",
      },
      createdAt: Math.floor(Date.now() / 1000),
      updatedAt: Math.floor(Date.now() / 1000),
    };

    mockStorage.load.mockReturnValue(mockSession);
    mockStorage.isExpired.mockReturnValue(false);

    await manager.initialize();

    expect(mockStorage.load).toHaveBeenCalled();
    expect(mockOAuthFlow.authenticate).not.toHaveBeenCalled();
    expect(manager.getClient()).toBeDefined();
  });

  it("should re-authenticate if session expired", async () => {
    const expiredSession: SessionData = {
      walletId: "wallet-123",
      organizationId: "org-456",
      authUserId: "user-789",
      accessToken: "access-token",
      refreshToken: "refresh-token",
      expiresAt: Math.floor(Date.now() / 1000) - 3600, // Expired
      stamperKeys: {
        publicKey: "public-key",
        secretKey: "secret-key",
      },
      createdAt: Math.floor(Date.now() / 1000),
      updatedAt: Math.floor(Date.now() / 1000),
    };

    mockStorage.load.mockReturnValue(expiredSession);
    mockStorage.isExpired.mockReturnValue(true);

    mockOAuthFlow.authenticate.mockResolvedValue({
      tokens: {
        access_token: "new-access-token",
        refresh_token: "new-refresh-token",
        expires_in: 3600,
      },
      walletId: "wallet-123",
      organizationId: "org-456",
      authUserId: "user-789",
    });

    await manager.initialize();

    expect(mockOAuthFlow.authenticate).toHaveBeenCalled();
    expect(mockStorage.save).toHaveBeenCalled();
  });

  it("should authenticate if no session exists", async () => {
    mockStorage.load.mockReturnValue(null);

    mockOAuthFlow.authenticate.mockResolvedValue({
      tokens: {
        access_token: "access-token",
        refresh_token: "refresh-token",
        expires_in: 3600,
      },
      walletId: "wallet-123",
      organizationId: "org-456",
      authUserId: "user-789",
    });

    await manager.initialize();

    expect(mockOAuthFlow.authenticate).toHaveBeenCalled();
    expect(mockStorage.save).toHaveBeenCalled();
  });
});
```

**Step 3: Run tests**

Run: `cd packages/mcp-server && yarn test`
Expected: All session manager tests pass

**Step 4: Commit**

```bash
git add packages/mcp-server/src/session/manager.ts packages/mcp-server/src/session/manager.test.ts
git commit -m "feat(mcp-server): add session manager with auto-auth

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 8: MCP Tools Implementation

**Files:**

- Create: `packages/mcp-server/src/tools/types.ts`
- Create: `packages/mcp-server/src/tools/list-wallets.ts`
- Create: `packages/mcp-server/src/tools/create-wallet.ts`
- Create: `packages/mcp-server/src/tools/sign-transaction.ts`
- Create: `packages/mcp-server/src/tools/sign-message.ts`
- Create: `packages/mcp-server/src/tools/index.ts`

**Step 1: Define tool types**

```typescript
import type { PhantomClient } from "@phantom/client";

export interface ToolContext {
  client: PhantomClient;
}

export interface ToolHandler<TParams = any, TResult = any> {
  name: string;
  description: string;
  inputSchema: {
    type: "object";
    properties: Record<string, any>;
    required?: string[];
  };
  handler: (params: TParams, context: ToolContext) => Promise<TResult>;
}
```

**Step 2: Implement list-wallets tool**

```typescript
import type { ToolHandler } from "./types";
import { Logger } from "../utils/logger";

const logger = new Logger("tool:list-wallets");

interface ListWalletsParams {
  limit?: number;
  offset?: number;
}

export const listWalletsTool: ToolHandler<ListWalletsParams> = {
  name: "list_wallets",
  description: "List wallets in the connected Phantom organization",
  inputSchema: {
    type: "object",
    properties: {
      limit: {
        type: "number",
        description: "Maximum number of wallets to return (default: 20)",
        minimum: 1,
        maximum: 100,
      },
      offset: {
        type: "number",
        description: "Number of wallets to skip for pagination (default: 0)",
        minimum: 0,
      },
    },
  },
  handler: async (params, context) => {
    try {
      logger.debug("Listing wallets", params);

      const result = await context.client.getWallets(params.limit || 20, params.offset || 0);

      logger.info(`Listed ${result.wallets.length} wallets`);

      return {
        wallets: result.wallets,
        totalCount: result.totalCount,
        limit: result.limit,
        offset: result.offset,
      };
    } catch (error: any) {
      logger.error("Failed to list wallets:", error);
      throw new Error(`Failed to list wallets: ${error.message}`);
    }
  },
};
```

**Step 3: Implement create-wallet tool**

```typescript
import type { ToolHandler } from "./types";
import { Logger } from "../utils/logger";

const logger = new Logger("tool:create-wallet");

interface CreateWalletParams {
  name?: string;
}

export const createWalletTool: ToolHandler<CreateWalletParams> = {
  name: "create_wallet",
  description: "Create a new wallet in the connected Phantom organization",
  inputSchema: {
    type: "object",
    properties: {
      name: {
        type: "string",
        description: "Optional name for the wallet",
      },
    },
  },
  handler: async (params, context) => {
    try {
      logger.debug("Creating wallet", params);

      const result = await context.client.createWallet(params.name);

      logger.info("Wallet created:", result.walletId);

      return {
        walletId: result.walletId,
        addresses: result.addresses,
      };
    } catch (error: any) {
      logger.error("Failed to create wallet:", error);
      throw new Error(`Failed to create wallet: ${error.message}`);
    }
  },
};
```

**Step 4: Implement sign-transaction tool**

```typescript
import type { ToolHandler } from "./types";
import { Logger } from "../utils/logger";

const logger = new Logger("tool:sign-transaction");

interface SignTransactionParams {
  walletId: string;
  transaction: string;
  networkId: string;
  derivationIndex?: number;
  account?: string;
}

export const signTransactionTool: ToolHandler<SignTransactionParams> = {
  name: "sign_transaction",
  description: "Sign a transaction with a Phantom wallet",
  inputSchema: {
    type: "object",
    properties: {
      walletId: {
        type: "string",
        description: "ID of the wallet to use for signing",
      },
      transaction: {
        type: "string",
        description: "Base64-encoded transaction to sign",
      },
      networkId: {
        type: "string",
        description: "Network ID (e.g., solana:mainnet, ethereum:mainnet)",
      },
      derivationIndex: {
        type: "number",
        description: "Optional derivation index (default: 0)",
        minimum: 0,
      },
      account: {
        type: "string",
        description: "Optional account address for simulation",
      },
    },
    required: ["walletId", "transaction", "networkId"],
  },
  handler: async (params, context) => {
    try {
      logger.debug("Signing transaction", { walletId: params.walletId, networkId: params.networkId });

      const result = await context.client.signTransaction({
        walletId: params.walletId,
        transaction: params.transaction,
        networkId: params.networkId as any,
        derivationIndex: params.derivationIndex,
        account: params.account,
      });

      logger.info("Transaction signed successfully");

      return {
        signedTransaction: result.rawTransaction,
      };
    } catch (error: any) {
      logger.error("Failed to sign transaction:", error);
      throw new Error(`Failed to sign transaction: ${error.message}`);
    }
  },
};
```

**Step 5: Implement sign-message tool**

```typescript
import type { ToolHandler } from "./types";
import { Logger } from "../utils/logger";
import { stringToBase64url } from "@phantom/base64url";
import { isEthereumChain } from "@phantom/utils";

const logger = new Logger("tool:sign-message");

interface SignMessageParams {
  walletId: string;
  message: string;
  networkId: string;
  derivationIndex?: number;
}

export const signMessageTool: ToolHandler<SignMessageParams> = {
  name: "sign_message",
  description: "Sign a message with a Phantom wallet",
  inputSchema: {
    type: "object",
    properties: {
      walletId: {
        type: "string",
        description: "ID of the wallet to use for signing",
      },
      message: {
        type: "string",
        description: "Message to sign (plain text)",
      },
      networkId: {
        type: "string",
        description: "Network ID (e.g., solana:mainnet, ethereum:mainnet)",
      },
      derivationIndex: {
        type: "number",
        description: "Optional derivation index (default: 0)",
        minimum: 0,
      },
    },
    required: ["walletId", "message", "networkId"],
  },
  handler: async (params, context) => {
    try {
      logger.debug("Signing message", { walletId: params.walletId, networkId: params.networkId });

      // Route to appropriate signing method based on network type
      const isEvm = isEthereumChain(params.networkId);

      const signature = isEvm
        ? await context.client.ethereumSignMessage({
            walletId: params.walletId,
            message: stringToBase64url(params.message),
            networkId: params.networkId as any,
            derivationIndex: params.derivationIndex,
          })
        : await context.client.signUtf8Message({
            walletId: params.walletId,
            message: params.message,
            networkId: params.networkId as any,
            derivationIndex: params.derivationIndex,
          });

      logger.info("Message signed successfully");

      return {
        signature,
      };
    } catch (error: any) {
      logger.error("Failed to sign message:", error);
      throw new Error(`Failed to sign message: ${error.message}`);
    }
  },
};
```

**Step 6: Create tool registry**

```typescript
import type { ToolHandler } from "./types";
import { listWalletsTool } from "./list-wallets";
import { createWalletTool } from "./create-wallet";
import { signTransactionTool } from "./sign-transaction";
import { signMessageTool } from "./sign-message";

export const tools: ToolHandler[] = [listWalletsTool, createWalletTool, signTransactionTool, signMessageTool];

export function getTool(name: string): ToolHandler | undefined {
  return tools.find(tool => tool.name === name);
}

export * from "./types";
```

**Step 7: Commit**

```bash
git add packages/mcp-server/src/tools/
git commit -m "feat(mcp-server): implement MCP tools for wallet operations

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 9: MCP Server Implementation

**Files:**

- Create: `packages/mcp-server/src/server.ts`
- Modify: `packages/mcp-server/src/index.ts`

**Step 1: Implement MCP server**

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { CallToolRequestSchema, ListToolsRequestSchema } from "@modelcontextprotocol/sdk/types.js";
import { Logger } from "./utils/logger";
import { SessionManager } from "./session/manager";
import { tools, getTool } from "./tools";
import type { ToolContext } from "./tools/types";

const logger = new Logger("Server");

export class PhantomMCPServer {
  private server: Server;
  private sessionManager: SessionManager;

  constructor() {
    // Create MCP server instance
    this.server = new Server(
      {
        name: "phantom-mcp-server",
        version: "0.1.0",
      },
      {
        capabilities: {
          tools: {},
        },
      },
    );

    // Initialize session manager
    this.sessionManager = new SessionManager();

    // Set up request handlers
    this.setupHandlers();
  }

  private setupHandlers(): void {
    // Handle tools/list request
    this.server.setRequestHandler(ListToolsRequestSchema, async () => {
      logger.debug("Handling tools/list request");

      return {
        tools: tools.map(tool => ({
          name: tool.name,
          description: tool.description,
          inputSchema: tool.inputSchema,
        })),
      };
    });

    // Handle tools/call request
    this.server.setRequestHandler(CallToolRequestSchema, async request => {
      const { name, arguments: args } = request.params;

      logger.debug("Handling tools/call request:", name);

      try {
        // Get tool handler
        const tool = getTool(name);
        if (!tool) {
          throw new Error(`Unknown tool: ${name}`);
        }

        // Get PhantomClient from session
        const client = this.sessionManager.getClient();

        // Create tool context
        const context: ToolContext = { client };

        // Execute tool
        const result = await tool.handler(args || {}, context);

        // Return success result
        return {
          content: [
            {
              type: "text",
              text: JSON.stringify(result, null, 2),
            },
          ],
        };
      } catch (error: any) {
        logger.error("Tool execution failed:", error);

        // Return error result
        return {
          content: [
            {
              type: "text",
              text: `Error: ${error.message}`,
            },
          ],
          isError: true,
        };
      }
    });
  }

  async start(): Promise<void> {
    try {
      logger.info("Starting Phantom MCP Server...");

      // Initialize session (authenticate if needed)
      await this.sessionManager.initialize();

      // Create stdio transport
      const transport = new StdioServerTransport();

      // Connect server to transport
      await this.server.connect(transport);

      logger.info("Phantom MCP Server started successfully");
      logger.info("Listening for MCP requests on stdio...");
    } catch (error) {
      logger.error("Failed to start server:", error);
      throw error;
    }
  }
}
```

**Step 2: Update main entry point**

```typescript
#!/usr/bin/env node

import { PhantomMCPServer } from "./server";

async function main() {
  const server = new PhantomMCPServer();
  await server.start();
}

main().catch(error => {
  console.error("[ERROR] Fatal error:", error);
  process.exit(1);
});
```

**Step 3: Build and test locally**

Run: `cd packages/mcp-server && yarn build`
Expected: Build succeeds

**Step 4: Test with echo**

Run: `echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | node packages/mcp-server/dist/index.js`
Expected: Returns list of available tools

**Step 5: Commit**

```bash
git add packages/mcp-server/src/server.ts packages/mcp-server/src/index.ts
git commit -m "feat(mcp-server): implement MCP server with stdio transport

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 10: Integration Testing

**Files:**

- Create: `packages/mcp-server/test/integration.test.ts`
- Create: `packages/mcp-server/README.md`

**Step 1: Create integration test**

```typescript
import { spawn } from "child_process";
import path from "path";

describe("MCP Server Integration", () => {
  const serverPath = path.join(__dirname, "../dist/index.js");

  it("should respond to tools/list request", done => {
    const server = spawn("node", [serverPath], {
      stdio: ["pipe", "pipe", "inherit"],
    });

    const request =
      JSON.stringify({
        jsonrpc: "2.0",
        id: 1,
        method: "tools/list",
      }) + "\n";

    let responseData = "";

    server.stdout.on("data", data => {
      responseData += data.toString();

      try {
        const response = JSON.parse(responseData);

        expect(response.jsonrpc).toBe("2.0");
        expect(response.id).toBe(1);
        expect(response.result.tools).toBeInstanceOf(Array);
        expect(response.result.tools.length).toBeGreaterThan(0);

        server.kill();
        done();
      } catch (e) {
        // Partial response, wait for more data
      }
    });

    server.stdin.write(request);
  }, 30000); // 30 second timeout for authentication
});
```

**Step 2: Create README**

````markdown
# @phantom/mcp-server

MCP (Model Context Protocol) server for Phantom wallet operations.

## Installation

### Global Installation

```bash
npm install -g @phantom/mcp-server
```
````

### Local Development

```bash
cd packages/mcp-server
yarn install
yarn build
```

## Usage

### With Claude Desktop

Add to your Claude Desktop configuration (`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS):

```json
{
  "mcpServers": {
    "phantom": {
      "command": "phantom-mcp"
    }
  }
}
```

### Manual Testing

Run the server manually:

```bash
phantom-mcp
```

Send a test request:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | phantom-mcp
```

## Authentication

On first run, the server will:

1. Automatically register an OAuth client with auth.phantom.app
2. Open your browser to connect.phantom.app for authorization
3. Start a local callback server on port 8080
4. Save your session to `~/.phantom-mcp/session.json`

Session persists across restarts and auto-refreshes when expired.

## Available Tools

### list_wallets

List wallets in your Phantom organization.

**Parameters:**

- `limit` (optional): Maximum number of wallets to return (default: 20)
- `offset` (optional): Number of wallets to skip (default: 0)

### create_wallet

Create a new wallet.

**Parameters:**

- `name` (optional): Name for the wallet

### sign_transaction

Sign a transaction with a wallet.

**Parameters:**

- `walletId`: ID of the wallet to use
- `transaction`: Base64-encoded transaction
- `networkId`: Network ID (e.g., "solana:mainnet", "ethereum:mainnet")
- `derivationIndex` (optional): Account derivation index (default: 0)
- `account` (optional): Account address for simulation

### sign_message

Sign a message with a wallet.

**Parameters:**

- `walletId`: ID of the wallet to use
- `message`: Plain text message to sign
- `networkId`: Network ID (e.g., "solana:mainnet", "ethereum:mainnet")
- `derivationIndex` (optional): Account derivation index (default: 0)

## Configuration

Environment variables:

- `DEBUG=1` or `PHANTOM_MCP_DEBUG=1`: Enable debug logging
- `PHANTOM_AUTH_BASE_URL`: Custom auth server URL (default: https://auth.phantom.app)
- `PHANTOM_CONNECT_BASE_URL`: Custom connect URL (default: https://connect.phantom.app)
- `PHANTOM_API_BASE_URL`: Custom API URL (default: https://api.phantom.app)
- `PHANTOM_CALLBACK_PORT`: Custom callback port (default: 8080)

## Security

- Session data is stored with secure file permissions (0600)
- All authentication uses OAuth 2.0 with PKCE
- Request signing uses Ed25519 keypairs
- Secrets are never logged

## Troubleshooting

### Browser doesn't open

If the browser doesn't open automatically, look for the authorization URL in the terminal output and open it manually.

### Port already in use

If port 8080 is in use, set `PHANTOM_CALLBACK_PORT` to a different port:

```bash
PHANTOM_CALLBACK_PORT=8081 phantom-mcp
```

### Session expired

The server automatically re-authenticates when the session expires. If you want to force re-authentication, delete the session file:

```bash
rm ~/.phantom-mcp/session.json
```

## Development

### Run tests

```bash
yarn test
```

### Watch mode

```bash
yarn dev
```

### Lint

```bash
yarn lint
```

## License

MIT

````

**Step 3: Run integration test**

Run: `cd packages/mcp-server && yarn test`
Expected: Integration test passes (may require manual auth on first run)

**Step 4: Commit**

```bash
git add packages/mcp-server/test/ packages/mcp-server/README.md
git commit -m "feat(mcp-server): add integration tests and documentation

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
````

---

## Task 11: Final Testing and Polish

**Files:**

- Verify all files are committed
- Test global installation
- Test with Claude Desktop

**Step 1: Test local link**

Run: `cd packages/mcp-server && npm link`
Expected: Creates global symlink to phantom-mcp

**Step 2: Test command**

Run: `phantom-mcp --help || phantom-mcp --version || echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | phantom-mcp`
Expected: Server responds correctly

**Step 3: Test with Claude Desktop (manual)**

1. Update Claude Desktop config
2. Restart Claude Desktop
3. Try asking Claude to list your Phantom wallets
4. Verify authentication flow works
5. Verify tool calls work correctly

**Step 4: Clean up**

Run: `npm unlink phantom-mcp -g`
Expected: Removes global symlink

**Step 5: Update root package.json workspaces**

Verify `packages/mcp-server` is included in workspaces array.

**Step 6: Final build**

Run: `cd packages/mcp-server && yarn build && yarn test`
Expected: All tests pass

**Step 7: Final commit**

```bash
git add packages/mcp-server/
git commit -m "feat(mcp-server): finalize MCP server implementation

- Global installation support
- OAuth/DCR authentication
- Session persistence
- Wallet operation tools
- Comprehensive error handling
- Integration tests
- Documentation

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Verification Checklist

After completing all tasks, verify:

- [ ] Package builds successfully: `yarn build`
- [ ] All tests pass: `yarn test`
- [ ] Global installation works: `npm link && phantom-mcp`
- [ ] OAuth flow completes successfully (browser opens, callback received)
- [ ] Session saved to `~/.phantom-mcp/session.json` with secure permissions
- [ ] Session persists across restarts
- [ ] All MCP tools work: list_wallets, create_wallet, sign_transaction, sign_message
- [ ] Error handling works correctly (invalid params, network errors)
- [ ] No stdout pollution (only JSON-RPC messages on stdout)
- [ ] Debug logging works with DEBUG=1
- [ ] Claude Desktop integration works

## Success Criteria

✅ Can install globally with `npm install -g @phantom/mcp-server`
✅ DCR successfully registers client with auth.phantom.app
✅ OAuth flow opens browser and handles callback
✅ Session persists across restarts
✅ All MCP tools work correctly
✅ No stdout pollution (only JSON-RPC messages)
✅ Works with Claude Desktop
✅ Comprehensive error handling and logging
