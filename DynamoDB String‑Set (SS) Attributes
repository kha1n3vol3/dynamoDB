# Using DynamoDB String‑Set (SS) Attributes to Store “Follows”

Examples in AWS Lambda (Node.js) + API Gateway

---

## 1. Why Choose an SS Attribute?

- **Unordered collection** – order doesn’t matter for “follows.”  
- **Uniqueness enforced** – you can’t add the same profileId twice.  
- **Atomic ADD / DELETE** – no read→modify→write; concurrent requests are safe.  
- **Space‑efficient** – one attribute per user (good up to hundreds of follows; ≳ thousands may hit the 400 KB item limit).

---

## 2. Table / Item Shape

**Primary Key**  
- `userId` (S) – ULID of the follower.

**Attribute**  
- `followedProfiles` (SS) – set of ULIDs this user follows.

Example low‑level item JSON:

```json
{
  "userId":           { "S": "01HZZ…ABC" },
  "followedProfiles": { "SS": ["01HZZ…DEF", "01HZZ…GHI"] }
}
```

CloudFormation / CDK attribute definition:

```yaml
Attributes:
  - AttributeName: userId
    AttributeType: S
  - AttributeName: followedProfiles
    AttributeType: SS
KeySchema:
  - AttributeName: userId
    KeyType: HASH
```

---

## 3. Common Operations (Atomic)

> Using the low‑level `@aws-sdk/client-dynamodb`.  
> If you use `DynamoDBDocumentClient` (v3) or `DocumentClient` (v2), swap `{ SS: [...] }` for `new Set([...])` or `docClient.createSet([...])`.

### A. Follow Somebody (ADD)

```js
await ddb.updateItem({
  TableName: process.env.FOLLOWS_TABLE,
  Key: { userId: { S: userId } },
  UpdateExpression: 'ADD followedProfiles :p',
  ExpressionAttributeValues: {
    ':p': { SS: [profileIdToFollow] }
  }
});
```

### B. Un‑follow Somebody (DELETE)

```js
await ddb.updateItem({
  TableName: process.env.FOLLOWS_TABLE,
  Key: { userId: { S: userId } },
  UpdateExpression: 'DELETE followedProfiles :p',
  ExpressionAttributeValues: {
    ':p': { SS: [profileIdToUnfollow] }
  }
});
```

### C. Read the List

```js
const res = await ddb.getItem({
  TableName: process.env.FOLLOWS_TABLE,
  Key: { userId: { S: userId } },
  ProjectionExpression: 'followedProfiles'
});
const follows = res.Item?.followedProfiles?.SS ?? [];
```

---

## 4. Node.js Lambda Helper (SDK v3 + DynamoDBDocumentClient)

```js
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import {
  DynamoDBDocumentClient,
  UpdateCommand,
  GetCommand
} from '@aws-sdk/lib-dynamodb';

const client = new DynamoDBClient({});
const doc    = DynamoDBDocumentClient.from(client); // auto‑marshalling

export async function follow(userId, profileId) {
  await doc.send(new UpdateCommand({
    TableName: process.env.FOLLOWS_TABLE,
    Key: { userId },
    UpdateExpression: 'ADD followedProfiles :p',
    ExpressionAttributeValues: {
      ':p': new Set([profileId])  // becomes SS
    }
  }));
}

export async function unfollow(userId, profileId) {
  await doc.send(new UpdateCommand({
    TableName: process.env.FOLLOWS_TABLE,
    Key: { userId },
    UpdateExpression: 'DELETE followedProfiles :p',
    ExpressionAttributeValues: {
      ':p': new Set([profileId])
    }
  }));
}

export async function listFollows(userId) {
  const res = await doc.send(new GetCommand({
    TableName: process.env.FOLLOWS_TABLE,
    Key: { userId },
    ProjectionExpression: 'followedProfiles'
  }));
  return Array.from(res.Item?.followedProfiles ?? []);
}
```

---

## 5. Optional Safety Guards

- **Cap follows per user**  
  ```js
  ConditionExpression: 'size(followedProfiles) < :max'
  ```
- **Prevent un‑following someone not followed**  
  ```js
  ConditionExpression: 'contains(followedProfiles, :p)'
  ```
- **Temporary follows**  
  Add a numeric `ttl` attribute + enable TTL on the table.

---

## 6. When **Not** to Use a Single SS

- A user follows so many profiles the item → 400 KB limit.  
- You need ordering (e.g. “latest follow first”).  
- You need the reverse edge (“who follows X?”) – store in a separate table or GSI.

> For the common “User → list of people they follow” edge, a DynamoDB SS is simple, cost‑effective, and performant.
