# Response to task by Ladunni Tegbe


**Questions**


- **At least three things you would improve**, and why.
- **The prompts and AI tools you used** to explore the app.
- **One thing AI got wrong or missed**, and how you noticed.


# Question 1
**At least three things you would improve**, and why.






# Question 2
**The prompts and AI tools you used** to explore the app.


**AI tools used**
*Claude*


**Prompt 1**
You're a senior React developer and my mentor. I'm a beginner learning React.


The code attached is a Task tracker app code.
Explain how each part of the code works: what each part does and how the pieces fit together.
Do not change any part of the code and keep each explanation for each part to at most 2 precise sentences.
Show each part you explain before explaining and keep explanations short and easy to understand.


**Prompt 2**


Now suggest three things that I could improve, and why.


Explain each suggestion properly


**Response**


Three Improvements
1. Handle errors and loading states


What's wrong now: Every fetch assumes the server always works. If the network drops or the server returns an error (like a 500), nothing happens: no message, no feedback, and in some cases the app can break. For example, res.json() is called without checking res.ok, so an error response gets treated as if it were a valid task and setTasks([task, ...tasks]) can put garbage into your list. There is also no "Loading..." message while the first fetch runs, so users see an empty table and may think they have no tasks.


How to improve it: Check res.ok before using the response, add a .catch() to every fetch (or use try/catch with async/await), and keep an error and a loading state to show messages.


jsx
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);


useEffect(() => {
  fetch("/api/tasks")
    .then((res) => {
      if (!res.ok) throw new Error("Failed to load tasks");
      return res.json();
    })
    .then((data) => setTasks(data))
    .catch((err) => setError(err.message))
    .finally(() => setLoading(false));
}, []);


Why it matters: Real apps fail often, and users need to know what happened. This is probably the single most important habit to build when working with APIs.


2. Use the functional form of state updates


What's wrong now: Several functions update tasks using the variable directly, like setTasks(tasks.map(...)) and setTasks([task, ...tasks]). These run inside .then(), which happens later, after the server replies. By then, tasks may be outdated (a "stale closure"). If you toggle two tasks quickly, the second response can overwrite the first one's update, because both used the old list.


How to improve it: Pass a function to the setter. React then gives you the latest value as prev.


jsx
// Instead of this:
setTasks(tasks.map((x) => (x.id === updated.id ? updated : x)));


// Do this:
setTasks((prev) => prev.map((x) => (x.id === updated.id ? updated : x)));


The same applies to addTask (setTasks((prev) => [task, ...prev])) and deleteTask (setTasks((prev) => prev.filter(...))).


Why it matters: This prevents subtle bugs that are very hard to find, because they only show up when actions happen close together. Using prev whenever the new state depends on the old state is a rule worth memorizing.


3. Split the big component into smaller ones, and move styles out


What's wrong now: Everything lives in one App component: seven state variables, all the API calls, the form, the filter buttons and the table. It's about 120 lines, which is manageable now, but it gets harder to read and change as you add features. Also, the inline style={{ ... }} objects are repeated and hard-coded (for example width: 900 is fixed, so the layout won't fit on a phone).


How to improve it: Break the UI into focused components, each with one job, and pass data down through props.


jsx
// Suggested structure
<App>
  <TaskForm onAdd={addTask} />
  <FilterBar filter={filter} setFilter={setFilter} remaining={remaining} />
  <TaskTable tasks={shown} onToggle={toggleDone} onEdit={saveEdit} onDelete={deleteTask} />
</App>


You could go further by moving the fetch logic into a custom hook like useTasks(), and moving the styling into a CSS file (or Tailwind, which you've worked with) so you can use responsive widths like max-width: 900px instead of a fixed 900.


Why it matters: Small components are easier to read, test and reuse, and this is how professional React projects are organized. It also helps you practice props, which is the core way React pieces talk to each other.


Suggested order: start with #1 (error handling), then #2 (a quick change), and do #3 last since it's the biggest refactor.


**Prompt 3**


Show me the former code and the new code you advised on improvement side by side


**Response**


# Before and After


I've stacked each pair (Before on top, After below) instead of placing them in columns, because code lines are wide and would wrap badly. The lines that changed are marked with comments.


---


## 1. Error and loading handling


**Before**
```jsx
const [tasks, setTasks] = useState([]);
// ...no loading or error state


useEffect(() => {
  fetch("/api/tasks")
    .then((res) => res.json())
    .then((data) => setTasks(data));
}, []);


function addTask(e) {
  e.preventDefault();
  fetch("/api/tasks", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ title: title, priority: priority, due_date: dueDate }),
  })
    .then((res) => res.json())
    .then((task) => {
      setTasks([task, ...tasks]);
      setTitle("");
      setPriority("medium");
      setDueDate("");
    });
}
```


**After**
```jsx
const [tasks, setTasks] = useState([]);
const [loading, setLoading] = useState(true);   // NEW
const [error, setError] = useState(null);       // NEW


useEffect(() => {
  fetch("/api/tasks")
    .then((res) => {
      if (!res.ok) throw new Error("Failed to load tasks");   // NEW: check response
      return res.json();
    })
    .then((data) => setTasks(data))
    .catch((err) => setError(err.message))                     // NEW: catch failures
    .finally(() => setLoading(false));                         // NEW: stop loading
}, []);


function addTask(e) {
  e.preventDefault();
  fetch("/api/tasks", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ title: title, priority: priority, due_date: dueDate }),
  })
    .then((res) => {
      if (!res.ok) throw new Error("Failed to add task");      // NEW
      return res.json();
    })
    .then((task) => {
      setTasks((prev) => [task, ...prev]);
      setTitle("");
      setPriority("medium");
      setDueDate("");
      setError(null);                                          // NEW: clear old error
    })
    .catch((err) => setError(err.message));                    // NEW
}
```


And in the JSX, right under the `<h1>`:
```jsx
{error && <p style={{ color: "red" }}>{error}</p>}
{loading && <p>Loading...</p>}
```


The other functions (`toggleDone`, `saveEdit`, `deleteTask`) get the same `res.ok` check and `.catch()`.


---


## 2. Functional state updates


**Before**
```jsx
// addTask
setTasks([task, ...tasks]);


// toggleDone
setTasks(tasks.map((x) => (x.id === updated.id ? updated : x)));


// saveEdit
setTasks(tasks.map((x) => (x.id === updated.id ? updated : x)));


// deleteTask
setTasks(tasks.filter((x) => x.id !== id));
```


**After**
```jsx
// addTask
setTasks((prev) => [task, ...prev]);


// toggleDone
setTasks((prev) => prev.map((x) => (x.id === updated.id ? updated : x)));


// saveEdit
setTasks((prev) => prev.map((x) => (x.id === updated.id ? updated : x)));


// deleteTask
setTasks((prev) => prev.filter((x) => x.id !== id));
```


---


## 3. Splitting into smaller components


**Before** (everything inside `App`'s `return`)
```jsx
return (
  <div style={{ width: 900, margin: 20, fontFamily: "Times New Roman" }}>
    <h1>Task Tracker</h1>


    <form onSubmit={addTask}>
      {/* title input, priority select, date input, Add button */}
    </form>


    <p>
      Show:{" "}
      {/* All / Active / Done buttons */}
      {" "}{remaining} left
    </p>


    <table border="1" cellPadding="4" style={{ width: 900, borderCollapse: "collapse" }}>
      {/* header + rows with checkbox, edit input, Edit/Delete buttons */}
    </table>
  </div>
);
```


**After**


`App` keeps the data and API functions and just assembles the pieces:
```jsx
return (
  <div style={{ maxWidth: 900, margin: 20, fontFamily: "Times New Roman" }}>
    <h1>Task Tracker</h1>
    <TaskForm onAdd={addTask} />
    <FilterBar filter={filter} setFilter={setFilter} remaining={remaining} />
    <TaskTable
      tasks={shown}
      onToggle={toggleDone}
      onSave={saveEdit}
      onDelete={deleteTask}
    />
  </div>
);
```


`TaskForm` owns the form inputs, since only the form needs them (`addTask` now receives the values instead of reading them from `App`):
```jsx
function TaskForm({ onAdd }) {
  const [title, setTitle] = useState("");
  const [priority, setPriority] = useState("medium");
  const [dueDate, setDueDate] = useState("");


  function handleSubmit(e) {
    e.preventDefault();
    onAdd({ title, priority, due_date: dueDate });
    setTitle("");
    setPriority("medium");
    setDueDate("");
  }


  return (
    <form onSubmit={handleSubmit}>
      <input value={title} onChange={(e) => setTitle(e.target.value)} placeholder="Task" />
      <select value={priority} onChange={(e) => setPriority(e.target.value)}>
        <option value="low">low</option>
        <option value="medium">medium</option>
        <option value="high">high</option>
      </select>
      <input type="date" value={dueDate} onChange={(e) => setDueDate(e.target.value)} />
      <button type="submit">Add</button>
    </form>
  );
}
```


`FilterBar` only displays buttons and a count:
```jsx
function FilterBar({ filter, setFilter, remaining }) {
  return (
    <p>
      Show:{" "}
      {["all", "active", "done"].map((f) => (
        <button
          key={f}
          onClick={() => setFilter(f)}
          style={{ fontWeight: filter === f ? "bold" : "normal" }}
        >
          {f}
        </button>
      ))}
      {" "}{remaining} left
    </p>
  );
}
```


`TaskTable` owns the editing state, since only the table needs it:
```jsx
function TaskTable({ tasks, onToggle, onSave, onDelete }) {
  const [editingId, setEditingId] = useState(null);
  const [editTitle, setEditTitle] = useState("");


  return (
    <table border="1" cellPadding="4" style={{ width: "100%", borderCollapse: "collapse" }}>
      <thead>
        <tr><th>Done</th><th>Task</th><th>Priority</th><th>Due</th><th>Actions</th></tr>
      </thead>
      <tbody>
        {tasks.map((t) => (
          <tr key={t.id}>
            <td><input type="checkbox" checked={!!t.done} onChange={() => onToggle(t)} /></td>
            <td style={{ textDecoration: t.done ? "line-through" : "none" }}>
              {editingId === t.id ? (
                <span>
                  <input value={editTitle} onChange={(e) => setEditTitle(e.target.value)} />
                  <button onClick={() => { onSave(t, editTitle); setEditingId(null); }}>Save</button>
                  <button onClick={() => setEditingId(null)}>Cancel</button>
                </span>
              ) : (
                t.title
              )}
            </td>
            <td style={{ color: t.priority === "high" ? "red" : t.priority === "medium" ? "orange" : "green" }}>
              {t.priority}
            </td>
            <td>{t.due_date}</td>
            <td>
              <button onClick={() => { setEditingId(t.id); setEditTitle(t.title); }}>Edit</button>
              <button onClick={() => onDelete(t.id)}>Delete</button>
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```


**Adjustments in `App` to match:**


- `addTask` now takes an object: `function addTask(newTask) { fetch(..., { body: JSON.stringify(newTask) }) ... }`
- `saveEdit` now takes the new title: `function saveEdit(t, newTitle) { ... title: newTitle ... }`
- `title`, `priority`, `dueDate`, `editingId` and `editTitle` are removed from `App`, since they now live in the child components.
- The fixed `width: 900` becomes `maxWidth: 900` (and `"100%"` on the table), so the layout can shrink on small screens.














# Question 3
**One thing AI got wrong or missed**, and how you noticed.
I am currently learning React and I honestly can't identify properly issues with React code by myself at the moment. I intend to study further to verify the response from both AI chats.

**I initially ChatGPT to check the improvements, I used this prompt**


const [tasks, setTasks] = useState([]);
const [loading, setLoading] = useState(true);   // NEW
const [error, setError] = useState(null);       // NEW


useEffect(() => {
  fetch("/api/tasks")
    .then((res) => {
      if (!res.ok) throw new Error("Failed to load tasks");   // NEW: check response
      return res.json();
    })
    .then((data) => setTasks(data))
    .catch((err) => setError(err.message))                     // NEW: catch failures
    .finally(() => setLoading(false));                         // NEW: stop loading
}, []);


function addTask(e) {
  e.preventDefault();
  fetch("/api/tasks", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ title: title, priority: priority, due_date: dueDate }),
  })
    .then((res) => {
      if (!res.ok) throw new Error("Failed to add task");      // NEW
      return res.json();
    })
    .then((task) => {
      setTasks((prev) => [task, ...prev]);
      setTitle("");
      setPriority("medium");
      setDueDate("");
      setError(null);                                          // NEW: clear old error
    })
    .catch((err) => setError(err.message));                    // NEW
}


work as a senior developer mentoring me, look through the code above and check to see any error. I asked for improvement on this code


const [tasks, setTasks] = useState([]);
// ...no loading or error state


useEffect(() => {
  fetch("/api/tasks")
    .then((res) => res.json())
    .then((data) => setTasks(data));
}, []);


function addTask(e) {
  e.preventDefault();
  fetch("/api/tasks", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ title: title, priority: priority, due_date: dueDate }),
  })
    .then((res) => res.json())
    .then((task) => {
      setTasks([task, ...tasks]);
      setTitle("");
      setPriority("medium");
      setDueDate("");
    });
}


Don't change the code, only show me the errors and then output your suggested corrections


**Response from ChatGPT**


As your senior developer mentor, I'll review this from two angles: **potential bugs in the improved code and whether the improvements address the weaknesses in your original code.**


Overall, the improvements are heading in the right direction. You've added loading states, error handling and safer state updates. However, there are a few things worth correcting.


## 1. Errors and issues in the improved code


### Issue 1: No `response.ok` check in the GET request?


You've addressed this correctly:


```jsx
if (!res.ok) throw new Error("Failed to load tasks");
```


**Why this matters:**


`fetch()` does not automatically reject a request when the server responds with an HTTP error such as 404 or 500.


Without checking `res.ok`, your application might attempt to process an error response as valid data.


Your improvement is correct.


### Issue 2: No validation of the returned data


You have:


```jsx
.then((data) => setTasks(data))
```


This assumes the API always returns an array.


If the API returns an unexpected response, such as an error object, your `tasks` state could contain something other than an array.


**Suggested correction:**


```jsx
.then((data) => {
  if (!Array.isArray(data)) {
    throw new Error("Invalid tasks response");
  }
  setTasks(data);
})
```


### Issue 3: Missing loading state in the POST request


You've introduced a loading state for fetching tasks, but not for adding a task.


This means a user can click the submit button multiple times while the first request is still processing.


**Suggested correction:**


Introduce a separate state:


```jsx
const [isSubmitting, setIsSubmitting] = useState(false);
```


Use it to prevent multiple submissions while a request is in progress.


### Issue 4: Potential stale state in `addTask`


Your improved code uses:


```jsx
setTasks((prev) => [task, ...prev]);
```


This is correct and is an improvement over your original code:


```jsx
setTasks([task, ...tasks]);
```


**Why?**


React state updates can be asynchronous.


Using the functional updater means React works with the latest available state rather than relying on the `tasks` value captured when the function was created.


Keep your improved version.


### Issue 5: Error state isn't cleared when fetching starts


You clear the error after successfully adding a task:


```jsx
setError(null);
```


However, you don't clear an existing error when a new GET request starts.


This can leave an old error message visible while another request is processing.


**Suggested correction:**


```jsx
setError(null);
setLoading(true);
```


Add these when initiating a new fetch, if the effect can run again.


### Issue 6: Missing cleanup for the GET request


Your `useEffect` doesn't handle component unmounting.


If the component unmounts before the request completes, the request continues running.


Consider using `AbortController` to cancel the request when the component unmounts.


---


## 2. Suggested corrections


Here is how I would approach the improvements while keeping your original structure.


### Improved GET request


```jsx
useEffect(() => {
  const controller = new AbortController();


  fetch("/api/tasks", { signal: controller.signal })
    .then((res) => {
      if (!res.ok) throw new Error("Failed to load tasks");
      return res.json();
    })
    .then((data) => {
      if (!Array.isArray(data)) {
        throw new Error("Invalid tasks response");
      }
      setTasks(data);
    })
    .catch((err) => {
      if (err.name !== "AbortError") {
        setError(err.message);
      }
    })
    .finally(() => {
      if (!controller.signal.aborted) {
        setLoading(false);
      }
    });


  return () => controller.abort();
}, []);
```


### Improved POST request


```jsx
function addTask(e) {
  e.preventDefault();


  setError(null);
  setIsSubmitting(true);


  fetch("/api/tasks", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      title,
      priority,
      due_date: dueDate,
    }),
  })
    .then((res) => {
      if (!res.ok) throw new Error("Failed to add task");
      return res.json();
    })
    .then((task) => {
      setTasks((prev) => [task, ...prev]);
      setTitle("");
      setPriority("medium");
      setDueDate("");
    })
    .catch((err) => setError(err.message))
    .finally(() => setIsSubmitting(false));
}
```


Remember to declare:


```jsx
const [isSubmitting, setIsSubmitting] = useState(false);
```


---


## 3. My senior developer assessment


| Area | Assessment |
|---|---|
| Loading state | Good improvement |
| Error handling | Good improvement |
| HTTP response checking | Correct |
| Functional state update | Correct |
| Error clearing | Needs improvement |
| Data validation | Missing |
| Request cancellation | Missing |
| POST submission state | Missing |


### Three things I want you to understand


As you continue learning React, pay attention to these concepts:


1. **`res.ok`:** Checking whether an HTTP request succeeded. Remember that `fetch()` doesn't reject automatically for HTTP errors.


2. **Functional state updates:** Understand why `setTasks(prev => ...)` is safer than `setTasks([...tasks])` when the next state depends on the previous state.


3. **`AbortController`:** Learn how to cancel requests and prevent unnecessary work when components unmount.


**My recommendation:** Don't just copy the corrections. Try explaining why each change is necessary. That's how you move from knowing React syntax to understanding how React applications behave in real projects.
