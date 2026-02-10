# 🌳 Three-Generation PSID Family Tree Explorer

**Longitudinal Study: 1968-Present**

Explore how **1968 homeownership** affected educational outcomes measured decades later:

- **Generation 1 (Blue/Purple)**: Household heads in 1968 - Homeownership status
- **Generation 2 (Colored)**: Their children - Education measured as adults
- **Generation 3 (Colored)**: Grandchildren - Education measured as adults (1990s-2020s)

**Color = Education Level** | **Shape = Homeownership**

This shows the **long-term intergenerational effects** of homeownership!


```python
# Install required packages
!pip install networkx matplotlib ipywidgets -q
```

    [?25l   [90m━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━[0m [32m0.0/1.6 MB[0m [31m?[0m eta [36m-:--:--[0m[2K   [91m━━━━━━━━━━━[0m[90m╺[0m[90m━━━━━━━━━━━━━━━━━━━━━━━━━━━━[0m [32m0.5/1.6 MB[0m [31m14.0 MB/s[0m eta [36m0:00:01[0m[2K   [91m━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━[0m[91m╸[0m [32m1.6/1.6 MB[0m [31m31.1 MB/s[0m eta [36m0:00:01[0m[2K   [90m━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━[0m [32m1.6/1.6 MB[0m [31m20.2 MB/s[0m eta [36m0:00:00[0m
    [?25h


```python
# Import libraries
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import networkx as nx
from ipywidgets import interact, widgets, HBox, VBox, Button, Output
from IPython.display import display, clear_output
import warnings
warnings.filterwarnings('ignore')

print("✅ Libraries imported successfully!")
```

    ✅ Libraries imported successfully!



```python
# Mount Google Drive
from google.colab import drive
drive.mount('/content/drive')
print("✅ Google Drive mounted!")
```

    Mounted at /content/drive
    ✅ Google Drive mounted!



```python
# Load data
DATA_PATH = '/content/drive/MyDrive/DATA/PSID_data/analysis_ready_data_WITH_PARENT_ED.csv'

print("📂 Loading data...")
df = pd.read_csv(DATA_PATH)
print(f"✅ Loaded {len(df):,} rows")
print(f"\n📊 Key columns: {[c for c in df.columns if 'parent' in c or 'grand' in c or 'child' in c][:10]}")
```

    📂 Loading data...
    ✅ Loaded 59,626 rows
    
    📊 Key columns: ['grandparent_id', 'parent_id', 'parent_homeowner', 'parent_sex', 'parent_race', 'child_sex', 'child_race', 'child_education_years', 'child_education_level', 'parent_head_education']



```python
# Data preparation for 3-generation trees
print("🔧 Preparing 3-generation data...")

# Only require grandparent columns to be complete
grandparent_cols = ['grandparent_id', 'parent_homeowner']
df_clean = df.dropna(subset=grandparent_cols)

print(f"After filtering for complete grandparent data: {len(df_clean):,} rows")

# Create mappings
race_map = {1: 'White', 2: 'Black', 3: 'Hispanic', 4: 'Other'}
homeowner_map = {1.0: 'Owner', 0.0: 'Renter'}
sex_map = {1.0: 'Male', 2.0: 'Female'}

# Apply mappings for all generations
df_clean['grandparent_homeowner_label'] = df_clean['parent_homeowner'].map(homeowner_map)

# Grandparent race - use parent_race as proxy for grandparent race
if 'parent_race' in df.columns:
    df_clean['grandparent_race_label'] = df_clean['parent_race'].map(race_map)
    df_clean['parent_race_label'] = df_clean['parent_race'].map(race_map)

# Check for parent education columns
if 'parent_max_education' in df.columns:
    parent_ed_map = {
        1: '0-5 grades', 2: '6-8 grades', 3: '9-11 grades',
        4: 'HS Graduate', 5: 'Some College', 6: 'College+',
        7: 'Advanced Degree', 8: 'Unknown'
    }
    df_clean['parent_ed_label'] = df_clean['parent_max_education'].map(parent_ed_map)

# Child data
if 'child_sex' in df.columns:
    df_clean['child_sex_label'] = df_clean['child_sex'].map(sex_map)

# Child race
if 'child_race' in df.columns:
    df_clean['child_race_label'] = df_clean['child_race'].map(race_map)

# Check if we have child homeownership data
child_owner_col = None
for col in df.columns:
    if 'child' in col.lower() and 'owner' in col.lower():
        child_owner_col = col
        df_clean['child_homeowner_label'] = df_clean[col].map(homeowner_map)
        print(f"   Found child homeownership column: {col}")
        break

# Categorize education for all generations
def categorize_ed(years):
    if pd.isna(years):
        return 'Unknown'
    elif years < 12:
        return '<HS'
    elif years == 12:
        return 'HS'
    elif years < 16:
        return 'Some College'
    elif years >= 16:
        return 'College+'
    return 'Unknown'

if 'child_education_years' in df.columns:
    df_clean['child_ed_category'] = df_clean['child_education_years'].apply(categorize_ed)

# Count structure
g1_counts = df_clean['grandparent_id'].nunique()
g2_counts = df_clean['parent_id'].nunique()
g3_counts = len(df_clean)

print(f"\n📈 Three-Generation Structure:")
print(f"   Generation 1 (Grandparents): {g1_counts:,} families")
print(f"   Generation 2 (Parents): {g2_counts:,} families")
print(f"   Generation 3 (Children): {g3_counts:,} individuals")

# Show race distribution
if 'grandparent_race_label' in df_clean.columns:
    print(f"\n📊 Race distribution:")
    race_dist = df_clean.groupby('grandparent_race_label')['grandparent_id'].nunique()
    for race, count in race_dist.items():
        print(f"   {race}: {count:,} families")

# Create family IDs
unique_grandparents = df_clean['grandparent_id'].unique()
family_id_map = {gid: f"Family_{i:05d}" for i, gid in enumerate(unique_grandparents, 1)}
df_clean['family_id'] = df_clean['grandparent_id'].map(family_id_map)

print(f"\nAvailable data columns: {[c for c in df_clean.columns if 'label' in c or 'category' in c]}")
print(f"\n✅ Data preparation complete!")
```

    🔧 Preparing 3-generation data...
    After filtering for complete grandparent data: 39,033 rows
    
    📈 Three-Generation Structure:
       Generation 1 (Grandparents): 14,453 families
       Generation 2 (Parents): 33,450 families
       Generation 3 (Children): 39,033 individuals
    
    📊 Race distribution:
       Black: 4,233 families
       Hispanic: 628 families
       White: 9,461 families
    
    Available data columns: ['grandparent_homeowner_label', 'grandparent_race_label', 'parent_race_label', 'parent_ed_label', 'child_sex_label', 'child_race_label', 'child_ed_category']
    
    ✅ Data preparation complete!



```python
# Build 3-generation tree
def build_three_gen_tree(family_rows):
    """
    Build a 3-generation family tree showing educational outcomes.

    Research Question: Does Gen1 homeownership affect Gen2 & Gen3 education?

    Gen 1 = Grandparent (1968 homeowner status)
    Gen 2 = Parent (has parent_max_education)
    Gen 3 = Child (has child_education_years - surveyed as adults)
    """
    first_row = family_rows.iloc[0]

    tree = {
        'family_id': first_row['family_id'],
        'g1': {
            'id': first_row['grandparent_id'],
            'race': first_row.get('grandparent_race_label', 'Unknown'),
            'homeowner_1968': first_row.get('grandparent_homeowner_label', 'Unknown'),
        },
        'g2': {},  # Will be dict of parent_id -> parent data
        'g3': []   # List of all grandchildren
    }

    # Group by parent_id to get Generation 2 structure
    for parent_id in family_rows['parent_id'].unique():
        parent_rows = family_rows[family_rows['parent_id'] == parent_id]
        first_parent_row = parent_rows.iloc[0]

        # Gen 2 education (parent_max_education)
        tree['g2'][parent_id] = {
            'id': parent_id,
            'race': first_parent_row.get('parent_race_label', 'Unknown'),
            'education_category': first_parent_row.get('parent_ed_label', 'Unknown'),
            'education_code': first_parent_row.get('parent_max_education'),  # 1-8 code
            'homeowner': first_parent_row.get('parent_homeowner_label', 'Unknown'),
            'children': []
        }

        # Gen 3 - Add each child with their education
        for idx, row in parent_rows.iterrows():
            child_data = {
                'parent_id': parent_id,
                'sex': row.get('child_sex_label', 'Unknown'),
                'race': row.get('child_race_label', 'Unknown'),
                'education_years': row.get('child_education_years'),  # Actual years
                'education_category': row.get('child_ed_category', 'Unknown'),
                'homeowner': row.get('child_homeowner_label', 'Unknown'),
            }
            tree['g2'][parent_id]['children'].append(child_data)
            tree['g3'].append(child_data)

    return tree

print("✅ Three-generation tree builder defined")
```

    ✅ Three-generation tree builder defined



```python
# Visualize 3-generation tree with CLEAR education labels
def visualize_three_gen_tree(tree, figsize=(18, 12)):
    """
    Visualize educational outcomes across 3 generations.
    COLOR = Education level, SHAPE = Homeownership
    """
    G = nx.DiGraph()

    def get_education_color_from_years(years):
        """Color based on years of education."""
        if pd.isna(years):
            return '#90A4AE'  # Gray
        if years < 12:
            return '#D32F2F'  # Red
        elif years == 12:
            return '#F57C00'  # Orange
        elif years < 16:
            return '#FBC02D'  # Yellow
        elif years == 16:
            return '#388E3C'  # Green
        else:
            return '#1976D2'  # Blue

    def get_education_color_from_code(code):
        """Color based on parent_max_education codes (1-8)."""
        if pd.isna(code):
            return '#90A4AE'
        code = int(code)
        if code <= 3:  # 0-11 grades
            return '#D32F2F'
        elif code == 4:  # HS
            return '#F57C00'
        elif code == 5:  # Some College
            return '#FBC02D'
        elif code == 6:  # College
            return '#388E3C'
        elif code >= 7:  # Advanced
            return '#1976D2'
        return '#90A4AE'

    def get_ed_label_from_code(code):
        """Get SHORT clear label from code."""
        if pd.isna(code):
            return '?'
        code = int(code)
        if code <= 3:
            return '<HS'
        elif code == 4:
            return 'HS'
        elif code == 5:
            return 'SomeColl'
        elif code == 6:
            return 'College'
        elif code >= 7:
            return 'Grad+'
        return '?'

    def get_ed_label_from_years(years):
        """Get SHORT clear label from years."""
        if pd.isna(years):
            return '?'
        y = int(years)
        if y < 12:
            return f'{y}y (<HS)'
        elif y == 12:
            return '12y (HS)'
        elif y < 16:
            return f'{y}y (SomeColl)'
        elif y == 16:
            return '16y (College)'
        else:
            return f'{y}y (Grad+)'

    def get_ownership_shape(owner_status):
        if owner_status == 'Owner':
            return 'o'
        elif owner_status == 'Renter':
            return 's'
        else:
            return 'd'

    nodes_by_shape = {'o': [], 's': [], 'd': []}
    node_colors = {}
    node_labels = {}

    # Gen 1
    grandparent_node = 'g1'
    homeowner_status = tree['g1']['homeowner_1968']
    g1_color = '#7E57C2'
    g1_shape = get_ownership_shape(homeowner_status)
    g1_label = f"G1\n{tree['g1']['race']}\n{homeowner_status}"

    G.add_node(grandparent_node, label=g1_label, generation=1)
    nodes_by_shape[g1_shape].append(grandparent_node)
    node_colors[grandparent_node] = g1_color
    node_labels[grandparent_node] = g1_label

    # Gen 2
    parent_nodes = []
    for i, (parent_id, parent_data) in enumerate(tree['g2'].items()):
        parent_node = f"g2_{i}"
        parent_nodes.append(parent_node)

        parent_ed_code = parent_data.get('education_code')
        parent_owner = parent_data.get('homeowner', 'Unknown')

        parent_color = get_education_color_from_code(parent_ed_code)
        parent_shape = get_ownership_shape(parent_owner)

        # CLEAR label
        parent_label = f"G2-{i+1}\n{get_ed_label_from_code(parent_ed_code)}"

        G.add_node(parent_node, label=parent_label, generation=2)
        G.add_edge(grandparent_node, parent_node)

        nodes_by_shape[parent_shape].append(parent_node)
        node_colors[parent_node] = parent_color
        node_labels[parent_node] = parent_label

        # Gen 3
        for j, child in enumerate(parent_data['children']):
            child_node = f"g3_{i}_{j}"

            ed_years = child['education_years']
            child_color = get_education_color_from_years(ed_years)
            child_shape = get_ownership_shape(child.get('homeowner', 'Unknown'))

            # CLEAR label
            child_label = f"G3\n{child['sex']}\n{get_ed_label_from_years(ed_years)}"

            G.add_node(child_node, label=child_label, generation=3)
            G.add_edge(parent_node, child_node)

            nodes_by_shape[child_shape].append(child_node)
            node_colors[child_node] = child_color
            node_labels[child_node] = child_label

    # Layout
    pos = {}
    pos[grandparent_node] = (0, 2)

    num_parents = len(parent_nodes)
    if num_parents == 1:
        parent_x_positions = [0]
    else:
        parent_x_positions = np.linspace(-2, 2, num_parents)

    for i, parent_node in enumerate(parent_nodes):
        pos[parent_node] = (parent_x_positions[i], 1)

    for i, (parent_id, parent_data) in enumerate(tree['g2'].items()):
        num_children = len(parent_data['children'])
        parent_x = parent_x_positions[i]

        if num_children == 1:
            child_x_positions = [parent_x]
        else:
            spread = min(1.3, num_children * 0.4)
            child_x_positions = np.linspace(parent_x - spread/2, parent_x + spread/2, num_children)

        for j in range(num_children):
            child_node = f"g3_{i}_{j}"
            pos[child_node] = (child_x_positions[j], 0)

    # Plot
    fig, ax = plt.subplots(figsize=figsize)

    nx.draw_networkx_edges(G, pos, ax=ax, edge_color='#666666',
                          width=1.5, alpha=0.4, arrows=False)

    for shape, node_list in nodes_by_shape.items():
        if node_list:
            colors = [node_colors[n] for n in node_list]
            nx.draw_networkx_nodes(G, pos, nodelist=node_list,
                                  node_color=colors, node_size=5250,
                                  node_shape=shape, ax=ax,
                                  edgecolors='black', linewidths=2)

    nx.draw_networkx_labels(G, pos, node_labels, font_size=9,
                           font_color='white', font_weight='bold',
                           verticalalignment='center', ax=ax)

    # Title
    num_g2 = len(parent_nodes)
    num_g3 = len(tree['g3'])
    title = f"Family: {tree['family_id']}\n"
    title += f"G1 (1968): {tree['g1']['race']} {homeowner_status}"
    ax.set_title(title, fontsize=14, fontweight='bold', pad=20)

    # Legend
    from matplotlib.patches import Patch
    from matplotlib.lines import Line2D

    ed_legend = [
        Patch(facecolor='#D32F2F', label='<12y = <HS'),
        Patch(facecolor='#F57C00', label='12y = HS'),
        Patch(facecolor='#FBC02D', label='13-15y = Some College'),
        Patch(facecolor='#388E3C', label='16y = College'),
        Patch(facecolor='#1976D2', label='17+y = Graduate+'),
    ]

    shape_legend = [
        Line2D([0], [0], marker='o', color='w', markerfacecolor='gray',
               markersize=10, label='Owner', markeredgecolor='black'),
        Line2D([0], [0], marker='s', color='w', markerfacecolor='gray',
               markersize=10, label='Renter', markeredgecolor='black'),
        Line2D([0], [0], marker='d', color='w', markerfacecolor='gray',
               markersize=10, label='Unknown', markeredgecolor='black'),
    ]

    first_legend = ax.legend(handles=ed_legend, loc='upper left',
                            title='Years of Education', fontsize=8)
    ax.add_artist(first_legend)
    ax.legend(handles=shape_legend, loc='upper right',
             title='Homeownership', fontsize=8)

    ax.axis('off')
    plt.tight_layout()
    plt.show()

print("✅ Visualization function defined")
```

    ✅ Visualization function defined



```python
# Interactive widgets
print("🎨 Setting up interface...")

race_widget = widgets.Dropdown(
    options=['All', 'White', 'Black', 'Hispanic', 'Other'],
    value='All',
    description='Gen1 Race:',
    style={'description_width': '150px'},
    layout=widgets.Layout(width='350px')
)

homeowner_widget = widgets.Dropdown(
    options=['All', 'Owner', 'Renter'],
    value='All',
    description='Gen1 Homeowner (1968):',
    style={'description_width': '200px'},
    layout=widgets.Layout(width='400px')
)

search_widget = widgets.Text(
    placeholder='Enter Family ID (e.g., Family_00001)',
    description='Family ID:',
    style={'description_width': '100px'},
    layout=widgets.Layout(width='350px')
)

random_button = widgets.Button(
    description='🎲 Show Random Family',
    button_style='success',
    layout=widgets.Layout(width='300px', height='45px')
)

search_button = widgets.Button(
    description='🔍 Search by ID',
    button_style='info',
    layout=widgets.Layout(width='200px', height='45px')
)

output = Output()

print("✅ Widgets created!")
```

    🎨 Setting up interface...
    ✅ Widgets created!



```python
# Functions
def show_random_family(b=None):
    with output:
        clear_output(wait=True)

        filtered = df_clean.copy()

        # Apply race filter
        if race_widget.value != 'All':
            if 'grandparent_race_label' in filtered.columns:
                filtered = filtered[filtered['grandparent_race_label'] == race_widget.value]
                print(f"🔍 Filtered by race ({race_widget.value}): {filtered['grandparent_id'].nunique()} families")

        # Apply homeownership filter
        if homeowner_widget.value != 'All':
            filtered = filtered[filtered['grandparent_homeowner_label'] == homeowner_widget.value]
            print(f"🔍 Filtered by homeowner ({homeowner_widget.value}): {filtered['grandparent_id'].nunique()} families")

        if len(filtered) == 0:
            print("❌ No families found matching your criteria!")
            print("   Try relaxing some filters.")
            return

        # Pick random grandparent
        family_ids = filtered['family_id'].unique()
        random_family_id = np.random.choice(family_ids)

        family_rows = df_clean[df_clean['family_id'] == random_family_id]

        print(f"\n📊 Showing: {random_family_id}")
        print(f"   {len(family_rows)} total descendants in dataset")
        print(f"   {family_rows['parent_id'].nunique()} Gen2 parents")
        print("=" * 70)

        tree = build_three_gen_tree(family_rows)
        visualize_three_gen_tree(tree)

def search_by_id(b=None):
    with output:
        clear_output(wait=True)

        search_id = search_widget.value.strip()
        if not search_id:
            print("❌ Please enter a Family ID")
            return

        family_rows = df_clean[df_clean['family_id'] == search_id]

        if len(family_rows) == 0:
            print(f"❌ Family ID '{search_id}' not found")
            return

        print(f"\n📊 Showing: {search_id}")
        print("=" * 70)

        tree = build_three_gen_tree(family_rows)
        visualize_three_gen_tree(tree)

random_button.on_click(show_random_family)
search_button.on_click(search_by_id)

print("✅ Functions connected!")
```

    ✅ Functions connected!



```python
# Display interface
print("\n" + "="*80)
print("🌳 INTERGENERATIONAL EDUCATION EXPLORER (Longitudinal)")
print("="*80)
print("\nResearch Question: Does 1968 homeownership affect educational")
print("outcomes for children and grandchildren measured decades later?")
print("\nNote: Gen2 & Gen3 education measured when they were ADULTS.")
print("This shows COMPLETED education, not childhood education.\n")

filter_box = VBox([
    widgets.HTML("<h3>Filter by Gen1 (1968 Household)</h3>"),
    race_widget,
    homeowner_widget,
    random_button
])

search_box = VBox([
    widgets.HTML("<h3>Search Specific Family</h3>"),
    search_widget,
    search_button
])

top_section = HBox([filter_box, search_box])

display(top_section)
display(output)

print("\n✅ Filter by 1968 homeownership, see adult outcomes decades later!")
```

    
    ================================================================================
    🌳 INTERGENERATIONAL EDUCATION EXPLORER (Longitudinal)
    ================================================================================
    
    Research Question: Does 1968 homeownership affect educational
    outcomes for children and grandchildren measured decades later?
    
    Note: Gen2 & Gen3 education measured when they were ADULTS.
    This shows COMPLETED education, not childhood education.
    



    HBox(children=(VBox(children=(HTML(value='<h3>Filter by Gen1 (1968 Household)</h3>'), Dropdown(description='Ge…



    Output()


    
    ✅ Filter by 1968 homeownership, see adult outcomes decades later!

