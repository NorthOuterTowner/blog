# Utility Types

## Theory

### Partial<T>

**Principle:** Makes all properties of type `T` optional. Iterates over each property and adds the `?` modifier.

**Source Code:**
```typescript
type Partial<T> = {
  [P in keyof T]?: T[P];
};
```

---

### Required<T>

**Principle:** Makes all properties of type `T` required by removing the `?` modifier from each property.

**Source Code:**
```typescript
type Required<T> = {
  [P in keyof T]-?: T[P];
};
```

---

### Readonly<T>

**Principle:** Makes all properties of type `T` read-only by adding the `readonly` modifier to each property.

**Source Code:**
```typescript
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};
```

---

### Pick<T, K>

**Principle:** Constructs a type by selecting only properties `K` from type `T`. `K` must be a subset of `keyof T`.

**Source Code:**
```typescript
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};
```

---

### Omit<T, K>

**Principle:** Constructs a type by excluding properties `K` from type `T`. Combines `Pick` and `Exclude` to filter out unwanted keys.

**Source Code:**
```typescript
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
```

---

### Exclude<T, U>

**Principle:** Removes from `T` any type that is assignable to `U`. Uses distributive conditional types, applying the condition to each member of a union type individually.

**Source Code:**
```typescript
type Exclude<T, U> = T extends U ? never : T;
```

---

### Extract<T, U>

**Principle:** Extracts from `T` only those types that are assignable to `U`. The opposite of `Exclude`.

**Source Code:**
```typescript
type Extract<T, U> = T extends U ? T : never;
```

---

### NonNullable<T>

**Principle:** Removes `null` and `undefined` from union type `T`.

**Source Code:**
```typescript
type NonNullable<T> = T extends null | undefined ? never : T;
```

---

### Parameters<T>

**Principle:** Extracts the parameter types of function type `T` as a tuple. Uses conditional types with the `infer` keyword for type inference.

**Source Code:**
```typescript
type Parameters<T extends (...args: any) => any> = 
  T extends (...args: infer P) => any ? P : never;
```

---

### ReturnType<T>

**Principle:** Extracts the return type of function type `T`. Uses conditional types with the `infer` keyword for type inference.

**Source Code:**
```typescript
type ReturnType<T extends (...args: any) => any> = 
  T extends (...args: any) => infer R ? R : any;
```

---

### Record<K, T>

**Principle:** Creates an object type with keys of type `K` and values of type `T`. Used to build dictionaries or key-value mappings.

**Source Code:**
```typescript
type Record<K extends keyof any, T> = {
  [P in K]: T;
};
```

## Testing


```ts
/**
 * TypeScript 工具类型 (Utility Types) 完整测试文件
 * 运行方式: npx ts-node utility-types.test.ts
 * 或: npx tsx utility-types.test.ts
 */

// ============================================================
// 第一部分: 定义测试用的基础类型
// ============================================================

interface User {
  id: number;
  name: string;
  age: number;
  email?: string; // 可选属性
  role: "admin" | "user" | "guest";
}

// 用于 Exclude/Extract 测试的联合类型
type Status = "success" | "error" | "pending" | "timeout";

// 用于 Parameters/ReturnType 测试的函数
function createProduct(name: string, price: number, category: "electronics" | "book" | "food") {
  return {
    id: Math.floor(Math.random() * 10000),
    name,
    price,
    category,
    inStock: true,
  };
}

// ============================================================
// 第二部分: 工具类型测试 (每个测试都输出结果)
// ============================================================

console.log("=".repeat(60));
console.log("🧪 TypeScript 工具类型测试");
console.log("=".repeat(60));

// ---------- 1. Partial<T> ----------
console.log("\n📌 1. Partial<T> - 所有属性变为可选");
console.log("-".repeat(40));

type UserPartial = Partial<User>;
const partialUser: UserPartial = {
  name: "Alice", // 只传了 name, 其他属性都可选
  role: "user",
};
console.log("   Partial<User> 示例:", partialUser);
console.log("   类型: 所有属性都带 ?");
console.log("   ✅ 测试通过: 不需要提供全部属性");

// ---------- 2. Required<T> ----------
console.log("\n📌 2. Required<T> - 所有属性变为必填");
console.log("-".repeat(40));

type UserRequired = Required<User>;
// 注意: email 原本是可选的, 现在变成必填了
const requiredUser: UserRequired = {
  id: 1,
  name: "Bob",
  age: 25,
  email: "bob@example.com", // 必须提供
  role: "user",
};
console.log("   Required<User> 示例:", requiredUser);
console.log("   ✅ 测试通过: 所有属性都必须提供 (包括原本可选的 email)");

// ---------- 3. Readonly<T> ----------
console.log("\n📌 3. Readonly<T> - 所有属性变为只读");
console.log("-".repeat(40));

type UserReadonly = Readonly<User>;
const readonlyUser: UserReadonly = {
  id: 2,
  name: "Charlie",
  age: 30,
  role: "admin",
};
console.log("   Readonly<User> 示例:", readonlyUser);
// readonlyUser.name = "Charlie2"; // 取消注释会报错
console.log("   ✅ 测试通过: 属性不能被修改 (编译时会检查)");

// ---------- 4. Pick<T, K> ----------
console.log("\n📌 4. Pick<T, K> - 挑选部分属性");
console.log("-".repeat(40));

type UserPublic = Pick<User, "id" | "name" | "role">;
const publicUser: UserPublic = {
  id: 3,
  name: "Diana",
  role: "user",
};
console.log("   Pick<User, 'id' | 'name' | 'role'> 示例:", publicUser);
console.log("   ✅ 测试通过: 只包含选中的属性");

// ---------- 5. Omit<T, K> ----------
console.log("\n📌 5. Omit<T, K> - 排除部分属性");
console.log("-".repeat(40));

type UserSensitive = Omit<User, "email" | "age">;
const sensitiveUser: UserSensitive = {
  id: 4,
  name: "Eve",
  role: "admin",
};
console.log("   Omit<User, 'email' | 'age'> 示例:", sensitiveUser);
console.log("   ✅ 测试通过: 排除了 email 和 age");

// ---------- 6. Exclude<T, U> ----------
console.log("\n📌 6. Exclude<T, U> - 从联合类型中剔除");
console.log("-".repeat(40));

type StatusCompleted = Exclude<Status, "pending" | "timeout">;
// 结果: "success" | "error"
const completedStatus: StatusCompleted = "success";
console.log("   Exclude<Status, 'pending' | 'timeout'> 结果:", completedStatus);
console.log("   类型: 'success' | 'error'");
console.log("   ✅ 测试通过: 'pending' 和 'timeout' 被剔除了");

// ---------- 7. Extract<T, U> ----------
console.log("\n📌 7. Extract<T, U> - 从联合类型中提取");
console.log("-".repeat(40));

type StatusError = Extract<Status, "error" | "timeout">;
// 结果: "error" | "timeout"
const errorStatus: StatusError = "error";
console.log("   Extract<Status, 'error' | 'timeout'> 结果:", errorStatus);
console.log("   类型: 'error' | 'timeout'");
console.log("   ✅ 测试通过: 只提取了 'error' 和 'timeout'");

// ---------- 8. NonNullable<T> ----------
console.log("\n📌 8. NonNullable<T> - 剔除 null 和 undefined");
console.log("-".repeat(40));

type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>;
const definiteValue: DefiniteString = "Hello World";
// const invalidValue: DefiniteString = null; // 取消注释会报错
console.log("   NonNullable<string | null | undefined> 示例:", definiteValue);
console.log("   类型: string");
console.log("   ✅ 测试通过: null 和 undefined 被剔除了");

// ---------- 9. Parameters<T> ----------
console.log("\n📌 9. Parameters<T> - 提取函数参数类型 (元组)");
console.log("-".repeat(40));

type CreateProductParams = Parameters<typeof createProduct>;
// 结果: [string, number, "electronics" | "book" | "food"]
const params: CreateProductParams = ["Laptop", 999.99, "electronics"];
console.log("   Parameters<typeof createProduct> 示例:", params);
console.log("   类型: [string, number, 'electronics' | 'book' | 'food']");
console.log("   ✅ 测试通过: 参数类型被正确提取为元组");

// ---------- 10. ReturnType<T> ----------
console.log("\n📌 10. ReturnType<T> - 提取函数返回值类型");
console.log("-".repeat(40));

type CreateProductReturn = ReturnType<typeof createProduct>;
// 结果: { id: number; name: string; price: number; category: ...; inStock: boolean }
const productResult: CreateProductReturn = {
  id: 999,
  name: "Phone",
  price: 599,
  category: "electronics",
  inStock: true,
};
console.log("   ReturnType<typeof createProduct> 示例:", productResult);
console.log("   ✅ 测试通过: 返回值类型被正确提取");

// ---------- 11. Record<K, T> ----------
console.log("\n📌 11. Record<K, T> - 构建键值对类型");
console.log("-".repeat(40));

type Role = "admin" | "editor" | "viewer";
type RolePermissions = Record<Role, string[]>;
const permissions: RolePermissions = {
  admin: ["read", "write", "delete"],
  editor: ["read", "write"],
  viewer: ["read"],
};
console.log("   Record<Role, string[]> 示例:", permissions);
console.log("   ✅ 测试通过: 构建了角色->权限的映射");
```