# Object-Oriented Local-Storage Adapter

**Status:** Adapted excerpt. Imports, the body-conversion helper, comments, and the delete method were omitted for focus.

This class implements a provider-neutral storage interface. Its constructor injects the local root, its private path method applies the shared key policy, and its methods translate filesystem failures into typed application errors.

```ts
export class LocalStorage implements GeneralStorage {
  readonly #baseDir: string;

  constructor(baseDir: string) {
    this.#baseDir = path.resolve(baseDir);
  }

  #pathFor(key: string): string {
    assertValidObjectKey(key);
    return path.join(this.#baseDir, key);
  }

  async put(input: StoragePutInput): Promise<StoragePutResult> {
    const filePath = this.#pathFor(input.key);
    try {
      if (!(input.overwrite ?? false)) {
        const exists = await fs
          .access(filePath)
          .then(() => true)
          .catch(() => false);
        if (exists) {
          throw new StorageError(
            "put",
            `Object already exists at ${input.key} (overwrite not set).`,
          );
        }
      }

      await fs.mkdir(path.dirname(filePath), { recursive: true });
      await fs.writeFile(filePath, await toBuffer(input.body));
      return { key: input.key, ref: input.key };
    } catch (err) {
      if (err instanceof StorageError) throw err;
      throw new StorageError("put", `Local put failed for ${input.key}.`, err);
    }
  }

  async list(prefix: string, _cursor?: string): Promise<StorageListPage> {
    const items: StorageListItem[] = [];
    const walk = async (dir: string): Promise<void> => {
      const entries = await fs
        .readdir(dir, { withFileTypes: true })
        .catch(() => null);
      if (!entries) return;

      for (const entry of entries) {
        const abs = path.join(dir, entry.name);
        if (entry.isDirectory()) {
          await walk(abs);
        } else if (entry.isFile()) {
          const key = path
            .relative(this.#baseDir, abs)
            .split(path.sep)
            .join("/");
          if (key.startsWith(prefix)) items.push({ key, ref: key });
        }
      }
    };

    try {
      await walk(this.#baseDir);
    } catch (err) {
      throw new StorageError("list", `Local list failed for ${prefix}.`, err);
    }
    return { items, cursor: undefined };
  }
}
```

**Connected tests:** contract tests cover safe-key rejection, put/list/delete behaviour, overwrite refusal, Blob input, and adapter selection.

**Demonstrates:** object-oriented interface implementation, constructor injection, encapsulation, reusable error policy, and a testable adapter pattern.
