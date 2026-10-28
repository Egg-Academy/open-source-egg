# Shipping a tool

Use this when you add a tool to OpenSourceEgg. The folder you add is a design another person can build in their own app, with their own database, login, and hosting. It is not a copy of your app.

Read [CONTRIBUTING.md](./CONTRIBUTING.md), [GOVERNANCE.md](./GOVERNANCE.md), and [SECURITY.md](./SECURITY.md) as well. [nz-police-vetting/Police-Check-System.md](./nz-police-vetting/Police-Check-System.md) is an example of this shape.

## Folder

```
tool-name/
  README.md          what it is, and where to start
  SPEC.md            the design another coding assistant implements
  guide/             optional screenshots
```

Use a short folder name. One tool per folder. Do not put two products in one spec.

The README points at the spec and lists the files. The spec is the file an assistant follows. Name it `SPEC.md`, or a clearer name such as `Police-Check-System.md`.

## What the spec contains

Write behaviour, fields, and steps. Do not paste your framework.

1. **What it is.** Who it helps, and what it does not do.
2. **Out of scope.** Login, accounts, permissions, billing, and anything that belongs to the host app.
3. **Seams.** The few functions the host app already has. Name the inputs and outputs. Do not name a vendor.
4. **Records.** Fields, status names, and what causes each change. Plain language or a small table. Not a dump of your database.
5. **Steps.** What the person does, what the operator does, and what is stored.
6. **Checks.** What must be true before a step can finish.
7. **Their duties.** Privacy, employment, tax, or vetting rules stay with the person who runs the tool.

### Seams

A seam is a function the other app implements. Example:

```ts
requireOperator(): { operatorId: string }

sendMail(args: { to: string; subject: string; html: string }): Promise<void>

interface PrivateFiles {
  put(args: { key: string; bytes: Buffer; contentType: string }): Promise<void>
  signedGet(key: string, ttlSeconds: number): Promise<string>
  delete(key: string): Promise<void>
}
```

Use only the seams this tool needs. Say what they are for. Do not include your user table, session cookie, or file host.

You may add a short note titled **One way this was built**. A few lines. It is not the spec. The spec still has to work without that stack.

## Leave out

Do not commit:

- Real names, emails, phone numbers, or addresses
- Vetting results, identity documents, or children's private details
- Passwords, tokens, API keys, or `.env` files
- Production URLs, live file links, or hostnames from your running app
- Your whole database, login system, or app framework

Replace those with placeholders such as `Example Trust` and `https://app.example.org`.

Third-party forms stay under their own terms. Say that in the tool README. The NZ Police consent form is the example.

## Screenshot

A banner for the website is welcome. PNG, **1200×600** pixels (2:1). Show the whole screen, with no real personal data. The page centres it and does not crop it.

## Before you open the pull request

- [ ] A newcomer can build the tool from the spec without your repo
- [ ] Seams replace your database, login, email, and file storage
- [ ] No secrets, live records, or production URLs
- [ ] README in this repo lists the new folder
- [ ] You have the right to share the work under `LICENSE`
