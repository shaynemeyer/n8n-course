# Exercise 1: Hello World Workflow

## Objective

Create your first n8n workflow that demonstrates basic node connections and data flow.

**Time Required:** 15 minutes
**Difficulty:** Beginner
**Prerequisites:** n8n installed and running

---

## What You'll Learn

- How to add nodes to the canvas
- How to connect nodes together
- How to configure node settings
- How to execute a workflow
- How to view execution results

---

## Step-by-Step Instructions

### Step 1: Create a New Workflow

1. Open n8n in your browser (`http://localhost:5678`)
2. Click the "+ New workflow" button in the top left
3. You'll see an empty canvas with a prompt to add your first node

![New Workflow Canvas](../../../../diagrams/new-workflow-canvas.svg)

### Step 2: Add a Manual Trigger Node

The Manual Trigger node allows you to start the workflow manually with a button click.

1. Click the "+" button or "Add first step"
2. In the search box, type "Manual"
3. Click on "Manual Trigger" node
4. The node will appear on the canvas

**What it does:** This node starts the workflow when you click the "Execute Workflow" button.

### Step 3: Add a Set Node

The Set node allows you to add, modify, or remove data fields.

1. Hover over the Manual Trigger node
2. Click the "+" icon on the right side of the node
3. Search for "Set"
4. Select "Set" from the results
5. The Set node will be added and automatically connected

### Step 4: Configure the Set Node

Now let's add some data:

1. Click on the Set node to open its settings panel
2. Click "Add Value" button
3. Select "String" as the type
4. In the "Name" field, enter: `message`
5. In the "Value" field, enter: `Hello World!`
6. Click "Add Value" again
7. Select "Number" as the type
8. Name: `timestamp`
9. Value: `{{DateTime.now().toMillis()}}`

Your configuration should look like this:

```
Values to Set:
- String: message = "Hello World!"
- Number: timestamp = {{DateTime.now().toMillis()}}
```

### Step 5: Execute the Workflow

1. Click the "Execute Workflow" button in the top right
2. Watch as the workflow runs
3. You'll see checkmarks on the nodes indicating successful execution

### Step 6: View the Results

1. Click on the Set node after execution
2. In the right panel, you'll see the output data:

```json
{
  "message": "Hello World!",
  "timestamp": 1699564321789
}
```

**Success!** You've created and executed your first n8n workflow.

---

## Understanding What Happened

### Data Flow

```
Manual Trigger → Set Node
(Empty data)    (Added fields: message, timestamp)
```

1. **Manual Trigger**: Created an empty execution item
2. **Set Node**: Added two fields to the data
3. **Output**: A JSON object with your specified fields

### Key Concepts

- **Nodes** are connected left to right
- **Data flows** from one node to the next
- **Each node** processes and passes data to the next
- **Expressions** (like `{{DateTime.now()}}`) are evaluated dynamically

---

## Challenge: Extend Your Workflow

Now try these modifications on your own:

### Challenge 1: Add More Fields
Add these additional fields to the Set node:
- A boolean field called `isActive` with value `true`
- A string field called `author` with your name

### Challenge 2: Add Another Node
1. Add a second Set node after the first one
2. Configure it to add a field called `processed` with value `true`
3. Execute and view the combined output

### Challenge 3: Use Expressions
Modify the Set node to include:
- A field called `hour` with expression: `{{DateTime.now().hour}}`
- A field called `greeting` with expression: `{{$json.message}} from n8n!`

---

## Expected Output (After Challenges)

After completing all challenges, your output should look similar to:

```json
{
  "message": "Hello World!",
  "timestamp": 1699564321789,
  "isActive": true,
  "author": "Your Name",
  "processed": true,
  "hour": 14,
  "greeting": "Hello World! from n8n!"
}
```

---

## Workflow Diagram

```
┌─────────────────┐       ┌──────────────┐       ┌──────────────┐
│ Manual Trigger  │──────▶│  Set Node 1  │──────▶│  Set Node 2  │
│                 │       │              │       │              │
│ (Start)         │       │ + message    │       │ + processed  │
└─────────────────┘       │ + timestamp  │       └──────────────┘
                          │ + isActive   │
                          │ + author     │
                          └──────────────┘
```

---

## Troubleshooting

### Issue: Can't Find the Manual Trigger Node
**Solution:** Type "manual" in the search box when adding a node.

### Issue: Nodes Won't Connect
**Solution:** Click and drag from the small circle on the right side of a node to the left side of another node.

### Issue: Expression Not Working
**Solution:** Make sure expressions are wrapped in double curly braces: `{{expression}}`

### Issue: No Data Showing
**Solution:** Make sure you clicked "Execute Workflow" and then clicked on the node to see its output.

---

## Key Takeaways

- ✓ n8n workflows are visual representations of data flow
- ✓ Nodes are connected to pass data from left to right
- ✓ The Manual Trigger node is useful for testing
- ✓ The Set node can add or modify data fields
- ✓ Expressions in `{{}}` are evaluated dynamically
- ✓ Execution results can be viewed by clicking on nodes

---

## Next Steps

Great job! You've completed your first n8n workflow.

**Next Exercise:** [Exercise 2: Scheduled Email Notification](./exercise-2-scheduled-email.md)

**Or explore:**
- Try different node types from the node panel
- Read about [n8n expressions](https://docs.n8n.io/code-examples/expressions/)
- Check out [workflow examples](https://n8n.io/workflows)

---

## Save Your Workflow

Don't forget to save your work:
1. Click "Save" button in the top right
2. Give your workflow a name: "Hello World"
3. Optionally add tags: "learning", "tutorial"
4. Click "Save"

Congratulations on completing your first n8n workflow!
