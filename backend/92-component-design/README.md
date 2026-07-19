# Component Design Principles

You know that feeling when you open a component file and it's 800 lines long, handles data fetching, renders a UI, manages form state, and also somehow does animation logic? Yeah. We've all been there.

The problem isn't that the code works — it's that nobody can understand it, test it, or change it without breaking something. Good component design isn't about being fancy. It's about making sure future-you (or your teammate) doesn't curse your name at 2 AM.

## The One-Question Test

Here's a litmus test for any component: **can you describe what it does in one sentence?**

If you need two sentences, it might be doing too much.

```jsx
// ✅ Easy to describe: "Renders a user's avatar with an online/offline indicator"
<UserAvatar user={user} showStatus={true} />

// ❌ Hard to describe: "Renders a user profile that also fetches data, 
//     manages subscriptions, and handles payment history"
<UserProfileCard userId="42" />
```

This is the **Single Responsibility Principle** applied to components. A component should do one thing and do it well. When a component has multiple responsibilities, changes to one concern (say, data fetching) risk breaking another (say, layout).

## Props: The Public API of Your Component

Think of props as a function signature. Clear props make components easy to use without reading the implementation. Confusing props mean everyone has to open the file to figure out how to use it.

### Naming Matters

```jsx
// ❌ Ambiguous — what does `active` mean?
<Button active={true} />

// ❌ Conflicting types
<Button primary={true} secondary={false} />

// ✅ Clear intent
<Button variant="primary" disabled={false} />
```

### The Boolean Trap

Boolean props seem convenient, but they accumulate. What started as `<Button primary />` becomes a nightmare of combinations:

```jsx
// ❌ After 6 months of feature requests
<Button 
  primary 
  secondary 
  danger 
  large 
  rounded 
  outline 
  loading 
  disabled 
/>

// ✅ Single variant prop keeps it manageable
<Button 
  variant="primary" 
  size="lg" 
  shape="rounded"
  loading 
  disabled 
/>
```

### Prop Defaults Reduce Cognitive Load

```jsx
// ❌ Caller must know all the things
<Card 
  padding="md" 
  shadow="sm" 
  border={true} 
  radius="md" 
/>

// ✅ Sensible defaults — only override when different
<Card>
  <p>Most cards just need this.</p>
</Card>

// Still flexible when needed
<Card padding="lg" shadow="lg" radius="none">
  <p>Special card with custom spacing.</p>
</Card>
```

**Rule of thumb:** Provide defaults for 80% of use cases. The component should work out of the box with minimal props.

## Composition Over Configuration

There are two ways to make components flexible:

**Configuration-driven:** Pass a big config object that controls everything internally.

```jsx
// ❌ Configuration-driven — rigid, opaque
<DataTable 
  data={users} 
  columns={['name', 'email', 'role']} 
  sortable={true}
  filterable={true}
  paginated={true}
  pageSize={20}
  rowClick={(id) => navigate(`/users/${id}`)}
/>
```

**Composition-driven:** Let the caller compose smaller pieces.

```jsx
// ✅ Composition-driven — flexible, transparent
<DataTable data={users}>
  <DataTable.Column header="Name" accessor="name" />
  <DataTable.Column header="Email" accessor="email" />
  <DataTable.Column header="Role" accessor="role">
    {(role) => <RoleBadge role={role} />}
  </DataTable.Column>
  <DataTable.Pagination />
</DataTable>
```

Why composition wins:

| Concern | Configuration | Composition |
|---------|--------------|-------------|
| Adding new feature | Add more props (bloats API) | Add new sub-component (no API change) |
| Custom column rendering | Render prop or slot | Just use a different child |
| Layout control | Internal, hard to customize | Caller decides order/structure |
| Tree-shaking | Everything bundled | Only used sub-components bundled |
| Migration path | Breaking prop changes | Backward-compatible additions |

## Component Types — A Useful Mental Model

Not every component is the same. Grouping them by role helps decide what goes where.

### Presentational Components

These are your UI primitives. They don't know about data fetching, business logic, or app state. They receive data via props and fire events upward.

```jsx
// Pure presentational — no side effects, no data fetching
function UserCard({ name, email, avatarUrl, onEdit }) {
  return (
    <div className="user-card">
      <img src={avatarUrl} alt={name} className="avatar" />
      <div className="info">
        <h3>{name}</h3>
        <p className="email">{email}</p>
      </div>
      <button onClick={onEdit}>Edit</button>
    </div>
  );
}
```

**Characteristics:**
- Easy to test (pure output from props)
- Reusable across different contexts
- No dependencies on data layer
- Styling lives here

### Container / Connected Components

These handle the "how to get the data." They wrap presentational components and wire them to state, APIs, or event streams.

```jsx
// Container — fetches data, manages state, delegates rendering
function UserCardContainer({ userId }) {
  const [user, loading, error] = useUser(userId);

  if (loading) return <Skeleton />;
  if (error) return <ErrorState message={error.message} />;
  if (!user) return <EmptyState />;

  return (
    <UserCard
      name={user.name}
      email={user.email}
      avatarUrl={user.avatarUrl}
      onEdit={() => navigate(`/users/${userId}/edit`)}
    />
  );
}
```

**Characteristics:**
- Handles side effects (fetching, subscriptions)
- Manages loading/error/empty states
- Thin layer — mostly wiring
- Hard to unit test (but easy to integration test)

### Layout Components

These manage structure, not content. Think `<Grid>`, `<Sidebar>`, `<PageContainer>`.

```jsx
function PageLayout({ sidebar, main, header }) {
  return (
    <div className="page-layout">
      <header>{header}</header>
      <aside>{sidebar}</aside>
      <main>{main}</main>
    </div>
  );
}
```

The key: they don't care *what* children they receive, just *where* to put them.

## The Collocation Principle

Put things close to where they're used. Don't separate components, styles, and tests into separate folders mapped by type — that forces you to jump between 3 directories to understand one component.

```text
// ❌ Organized by type (forces context switching)
components/
├── Button.jsx
├── Button.css
├── Button.test.jsx
components/Header/
├── Header.jsx
├── Header.css
├── Header.test.jsx

// ✅ Organized by feature (everything in one place)
features/users/
├── UserCard.jsx       // component + inline styles
├── UserCard.test.jsx  // tests
├── useUser.js         // related hook
├── UserCard.stories.jsx // documentation
```

The benefit? When you're working on the user card feature, everything you need is in one directory. Deleting the feature means deleting one folder.

## Avoid Premature Abstraction

This is the biggest trap for mid-level devs. You see two components that look similar, so you extract a shared base component with 8 props and 4 conditional branches.

```jsx
// ❌ Premature abstraction
<FormField
  type="select"
  label="Country"
  options={countries}
  validation="required"
  layout="horizontal"
  size="md"
  helpText="Select your country"
  showIcon={true}
  iconPosition="left"
/>
```

Three months later, no one remembers what half those props do, and adding a new field variant breaks three other field types.

**Instead:** Let duplication be your guide. Don't abstract until you have at least **three** examples that share the same pattern. Two similar components might be coincidental. Three is a pattern.

## Reusability Guidelines

Not everything needs to be reusable. Some components are page-specific, and that's fine.

```jsx
// ✅ This is fine — tightly coupled to the Dashboard page
function DashboardWelcomeMessage({ userName }) {
  return (
    <div className="welcome-banner">
      <h1>Welcome back, {userName}!</h1>
      <p>Here's what happened since your last visit.</p>
    </div>
  );
}
```

The cost of making something reusable is real: more props, more edge cases, more testing, more documentation. Ask yourself: **will this component be used in more than one place?** If no, keep it simple.

## Error Boundaries and Edge Cases

Well-designed components handle the unexpected gracefully.

```jsx
function StarRating({ value = 0, max = 5, onChange }) {
  // Handle edge cases before rendering
  const safeValue = Math.max(0, Math.min(value, max));  // Clamp
  const safeMax = Math.max(1, max);                       // At least 1 star

  if (typeof onChange !== 'function') {
    console.warn('StarRating: onChange must be a function');
    return null;
  }

  return (
    <div className="star-rating" role="radiogroup" aria-label={`Rating ${safeValue} out of ${safeMax}`}>
      {Array.from({ length: safeMax }, (_, i) => (
        <Star
          key={i}
          filled={i < safeValue}
          onClick={() => onChange?.(i + 1)}
          aria-label={`${i + 1} star${i > 0 ? 's' : ''}`}
        />
      ))}
    </div>
  );
}
```

Things that *will* break if you don't plan for them:
- Null/undefined props
- Values outside expected range
- Missing callbacks or handlers
- Empty arrays or collections
- Long strings that break layout
- Loading and error states (every data-driven component needs them)

## Takeaways

- A component should do one thing — if you can't describe it in one sentence, split it up
- Props are your public API; keep them consistent, minimal, and self-explanatory
- Composition beats configuration for flexibility; prefer children over config objects
- Separate presentational (look) from container (data) concerns
- Collocate related files — put tests, styles, and hooks next to the component
- Don't abstract too early; three examples make a pattern, two is coincidence
- Plan for edge cases before they crash your app in production
