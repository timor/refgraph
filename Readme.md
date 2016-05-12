<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#orgheadline1">1. Idea</a></li>
<li><a href="#orgheadline34">2. Application</a>
<ul>
<li><a href="#orgheadline5">2.1. Styling</a>
<ul>
<li><a href="#orgheadline2">2.1.1. General CSS Classes</a></li>
<li><a href="#orgheadline3">2.1.2. Circle-Packing CSS Classes</a></li>
<li><a href="#orgheadline4">2.1.3. Tooltip</a></li>
</ul>
</li>
<li><a href="#orgheadline33">2.2. Code</a>
<ul>
<li><a href="#orgheadline6">2.2.1. Global Variables</a></li>
<li><a href="#orgheadline9">2.2.2. SVG DOM Structure</a></li>
<li><a href="#orgheadline10">2.2.3. Zooming</a></li>
<li><a href="#orgheadline28">2.2.4. Used Layouts</a></li>
<li><a href="#orgheadline31">2.2.5. View Modes</a></li>
<li><a href="#orgheadline32">2.2.6. Application Initialization/Data Loading</a></li>
</ul>
</li>
</ul>
</li>
<li><a href="#orgheadline35">3. Hacking</a></li>
</ul>
</div>
</div>


# Idea<a id="orgheadline1"></a>

-   Goal: get overview over certain scientific field by taking advantage
    of the information that is usually provided by reference managers,
    in addition to information that is usually provided by task planning systems.
-   extract information from bibtex files:
    -   author-paper relation
    -   number of citations per paper (custom field)
-   combine with:
    -   personal progress on papers
    -   personal classification on papers
-   several visualization modes
    1.  Collaboration Graph
    2.  Classification

# Application<a id="orgheadline34"></a>

The application basically consists of one html file that includes the
necessary CSS and Javascript.

It depends on d3.js for the
visualization, and on md5.js and a modified version of jdenticon for
identicon generation. 

    <!DOCTYPE html>
    <meta charset="utf-8">
    <style>
    
      <<css-definitions>>
    
    </style>
    <body>
      <script src="http://d3js.org/d3.v3.js"></script>
      <!-- <script src="scripts/jdenticon-1.3.2.js"></script> -->
      <script src="https://cdnjs.cloudflare.com/ajax/libs/blueimp-md5/2.3.0/js/md5.min.js"></script>
      <script>
    
        <<javascript-graph-functionality>>
    
      </script>
    </body>

## Styling<a id="orgheadline5"></a>

### General CSS Classes<a id="orgheadline2"></a>

General page layout:

    html, body { overflow: hidden }
    svg { position: fixed;
          top: 0;
          left: 0;
          width: 100%;
          height: 90%}

CSS Classes used:

-   **`.node`:** elements representing nodes (papers and authors)
-   **`.node_bg`:** the background of nodes (used for visualizing state
    information)
-   **`.link`:** the lines representing the links between nodes
-   **`.<state-class>`:** contains the actual specification for each state
    of a paper
-   **`.author`:** author nodes
-   **`.paper`:** paper nodes

All text in the nodes should use sans serif fonts, and be
non-selectable, because that interferes with dragging.  The background
of paper nodes is created by applying a [glow filter](#orgtarget1) to the rectangle
behind.  Depending on the personal state of the paper, different
colors are chosen.  The background of author nodes is hidden

    .node text {
        // pointer-events: none;
        font: 10px sans-serif;
        -webkit-user-select: none;  /* Chrome all / Safari all */
        -moz-user-select: none;     /* Firefox all */
        -ms-user-select: none;      /* IE 10+ */
        user-select: none;          /* Likely future */
    }
    
    .node_bg.paper {
        opacity: 0.2;
        filter: url(#glow);
    }
    
    .node_bg.paper.read {
        fill:green;
    }
    
    .node_bg.paper.unread {
        fill:red;
    }
    
    .node_bg.paper.started {
        fill:orange;
    }
    
    .node_bg.paper.overview {
        fill:yellow;
    }
    
    .node_bg.author {
        visibility: hidden;
    }
    
    .link {
        stroke: #ccc;
    }

### Circle-Packing CSS Classes<a id="orgheadline3"></a>

Specifically for the circle packing layout, which is currently used
for the Classification view:

-   **`.pack`:** elements inside the pack layout
-   **`.leaf`:** the leaf nodes of pack layout

All the circles in the pack layout are bluish, transparent and have a
thin border stroke.  The circles for the leaf nodes(the actual papers)
are not shown.

    circle.pack {
        fill: rgb(31,119,180);
        fill-opacity: .25;
        stroke-width: 1px;
    }
    
    .leaf {
        visibility: hidden;
    }
    
    text.pack {
        font: 12px sans-serif;
        stroke: #fcc;
        fill: #fcc;
    }

### Tooltip<a id="orgheadline4"></a>

The tooltip is styled here.

-   **`.tooltip_text`:** text of tooltips
-   **`.tooltip_bg`:** background (svg rect) of tooltips

    .tooltip_text {
        font: 12px sans-serif;
    }
    
    .tooltip_bg{
        fill: white;
        stroke: black;
        stroke-width: 1;
        opacity: 0.85;
    }

## Code<a id="orgheadline33"></a>

### Global Variables<a id="orgheadline6"></a>

For lack of better programming style, the following information is
defined in global variables:

    var width = 1200,               // width of the svg (not used correctly)
    
        height = 900,               // height of the svg portion (not used correctly)
    
        icon_size = 16,             // base size of icons for nodes
    
        jdenticon_size = 40        // base size of the identicons, note
                                    // that jdenticon does not allow
                                    // images smaller than 30, and padding
                                    // is added to that, so 40 should be a
                                    // safe minimum

For the imported json data, globals are defined for the top-level
elements of that data (TODO link to json data layout)

    var nodes, links, tree;

Other globals are defined before their respective usage.

### SVG DOM Structure<a id="orgheadline9"></a>

generally, d3.js functionality is used to generate the DOM structure.

The svg element should fill the whole width of the browser page, but
leave some space below for controls.  Also, pointer events have to be
caught explicitly.  These are actually later caught by the big background
rectangle (and I suppose bubbled to the svg element) to implement zooming and panning.

Note that the variable `svg` actually contains a `g` (group).

    var svg = d3.select("body").append("svg")
    // .attr("width", width)
    // .attr("height", height)
        .attr("height", "100%")
        .attr("width", "100%")
    // .attr("viewBox","-0 -250 700 500")
        .attr("pointer-events", "all")
        .append("g")
    // .attr("id","g1")
        .call(d3.behavior.zoom().on('zoom', redraw))
    ;

There is a transparent background rectangle for catching mouse events:

    svg.append("rect")
        .attr("width", width)
        .attr("height", height)
        .style("fill", "none")
    ;

There is a container group for all interactive content.  This is also
the one that the zoom and pan transformations are performed upon:

    var container = svg.append("g").attr("id","nodecontainer");

1.  Tooltips

    Tooltips appear when hovering over papers, showing the full title.
    
    There is only one tooltip consisting of a rect and text which live in the top group,
    and are placed as needed.
    
        var tooltip = svg.append("rect")
            .attr("class","tooltip_bg")
            .attr("id","tooltip_bg")
            .attr("visibility", "hidden")
            .attr("rx", 4)
            .attr("ry", 4)
            .attr("height",16)
            .attr("width",52);
        
        var tooltip_text = svg.append("text")
            .attr("class","tooltip_text")
            .attr("visibility", "hidden");
        
        function show_tooltip(d) {
            if (d.type == "paper") {
                x = d3.event.clientX;
                y = d3.event.clientY;
                tooltip_text
                    .text(d.labeltooltip)
                    .attr("visibility","visible")
                    .attr("x", x + 11)
                    .attr("y", y + 27);
                tooltip
                    .attr("visibility","visible")
                    .attr("x", x + 8)
                    .attr("y", y + 14)
                    .attr("width", tooltip_text.node().getComputedTextLength()+8);
            }
        }
        
        function hide_tooltip(d) {
            tooltip.attr("visibility", "hidden")
            tooltip_text.attr("visibility", "hidden")
        }
    
    The `show_tooltip` and `hide_tooltip` functions are later used as
    onMouseover and onMouseout handlers when the actual nodes are created
    (TODO: link)

2.  Filter for Node Background<a id="orgtarget1"></a>

    The blur effect of the node background is created here.  The defs node
    is attached directly to the `svg` DOM node.
    
        var defs = d3.select("svg").append("defs");
        var filter = defs.append("filter")
            .attr("id", "glow");
        filter.append("feGaussianBlur")
            .attr("stdDeviation", "3.5")
            .attr("result", "coloredBlur");

### Zooming<a id="orgheadline10"></a>

Zooming is provided as d3.js-provided behavior, with the following
being the zoom event handler.

    function redraw() {
        container.attr("transform", "translate(" + d3.event.translate + ")scale(" + d3.event.scale +")");
        // svg.attr("transform", "translate(" + d3.event.translate + ")");
    };

### Used Layouts<a id="orgheadline28"></a>

Several different d3.js layouts are used.  All of them are initialized
here.  For some reason it is important that the force layout is
created last.  Also, the initial mode is set to the collaboration
layout.

    function make_layout() {
        connect_node(tree);
    
        make_pack_layout();
    
        make_force_layout();
    
        change_mode('collaboration');
    
    }

1.  Force Layout

    The force layout is used to duisplay the collaboration graph.
    All the global properties are set when creating the initial `force`
    object.  Interactive aspects of the layout are handled in
    [`change_mode`](#orgsrcblock1).
    
    For different modes, different settings are used for the following
    global variables:
    
        var kx_mul = 0.15,              // multiplier for attractor force in x direction
        
            ky_mul = 0.4,               // multiplier for attractor force in y direction
        
            node_charge_mul = 1;        // multiplier for node charge
    
    Gravity is turned off because all paper nodes have an attractor, so
    the layout does face the danger of expanding indefinitely.  Charge
    Distance is set, but it seems it does not have a notable influence on
    performance.  It seems because charges are quite high, friction was
    "increased" from the default 0.9 to 0.7 to stop high-speed movement.
    
        var force = d3.layout.force()
            .gravity(0)
            .distance(50)
            .chargeDistance(800)
            .friction(0.7)
            .size([width, height]);
    
    Here is the force layout initialization.  It must be called after data is
    available.  See [2.2.4.1.3](#orgtarget2) for what actually happens, and [2.2.4.1.4](#orgtarget3)
    for the tick event handler that is attached.
    
        function make_force_layout() {
            var link,                   // selection of created svg elements for link representation
        
                node                    // selection of created svg elements for node representation
        
            <<force-layout-initialization>>
        
            <<force-tick-handler>>
        }
    
    1.  Node Property Helper functions
    
        Several node properties are data-dependent.  The following definitions
        are used to calculate the relevant values for the layout.
        
        1.  Node Significance
        
            Used as basis for other layout properties.
            
            The significance of authors is determined by the balls they have, and
            weighted using a fractional-exponent exponential function, to be able
            to distinguish the less-significant authors better, since there are
            usually more of them.
            
            The significance of papers is the number of citations they have.  This
            is weighted logarithmically for similar reasons.
            
                function node_significance(d) {
                    if (d.type == "author")
                        // return icon_size * (1 + (d.balls/20);
                        return (1 + (Math.pow((d.balls-1), 0.8) * 0.5));
                    else
                        return (1 + Math.log10(1 + d.citations));
                }
        
        2.  Node Image Positioning
        
                function node_image_size(d) {
                    return icon_size * node_significance(d);
                };
            
            Used to center the image for a node.
            
                function node_image_offset(d) {
                    return - (node_image_size(d) / 2);
                }
        
        3.  Node Charge
        
            For the collaboration layout, the node charge is made dependent on the
            node significance.  This way, it is easier to place lesser-significant
            nodes around the more central nodes.
            
                function collab_charge(d) {
                    return (node_significance(d) * -250);
                }
    
    2.  Node Dragging Behaviour
    
        Dragging is provided by a d3.js behavior, but the default event
        handlers are not used.
        
            var drag = d3.behavior.drag()
                .origin(function(d) { return d; })
                .on("dragstart", dragstarted)
                .on("drag", dragged)
                .on("dragend", dragended);
        
        Instead, the following handlers are implemented.  Note that they rely
        on undocumented internals (the meaning of the individual bits of the
        `fixed` property).  These are copied from the original functions.
        
            function dragstarted(d) {
                d3.event.sourceEvent.stopPropagation();
                d3.select(this).classed("dragging", true);
                // force.d3_layout_forceDragstart(d);
                d.fixed |= 2; // set bit 2
            }
        
            function dragged(d) {
                // d3.select(this).attr("cx", d.x = d3.event.x).attr("cy", d.y = d3.event.y);
                // d.x = d3.event.x, d.y = d3.event.y;
                d.px = d3.event.x, d.py = d3.event.y;
                force.resume(); // restart annealing
            }
        
            function dragended(d) {
                d3.select(this).classed("dragging", false);
                // force.d3_layout_forceDragend(d);
                d.fixed &= ~6; // unset bits 2 and 3
            }
    
    3.  Force Layout Initialization <a id="orgtarget2"></a>
    
        1.  Connecting Layout to Data
        
            Feed the force layout with the actual data.  d3.js expects a certain
            data layout, from which it initializes connectivity and node
            properties (TODO: link)
            
                force
                    .nodes(nodes)
                    .links(links)
                ;
        
        2.  Creating the SVG elements
        
            d3.js's enter selection mechanism is used to get the actually created
            svg DOM nodes for the links (lines) and the nodes (groups).
            
                link = container.selectAll(".link")
                    .data(links)
                    .enter().append("line")
                    .attr("class", "link");
                
                node = container.selectAll(".node")
                    .data(nodes)
                    .enter().append("g")
                    .attr("class", "node")
                    .on("mouseover", show_tooltip)
                    .on("mouseout", hide_tooltip)
                    .call(drag);
        
        3.  Node Background
        
            The background rectangles are attached.  These are connected to the
            [2.2.2.2](#orgtarget1) using [CSS](#orgheadline5).  A class is added with the same name as the
            personal reading state indicated in the data.
            
                node.append("rect")
                    .attr("x", node_image_offset)
                    .attr("y", node_image_offset)
                    .attr("width", node_image_size)        
                    .attr("height", node_image_size)
                    .attr("class", function(d) {
                        var s= "node_bg " + d.type;
                        if (d.type == "paper") s = s + " " + d.state;
                        return s;
                    });
        
        4.  Node Images
        
            The nodes themselves are represented by images.  Depending on the node
            type, different images are loaded.
            
                node.append("image")
                    .attr("xlink:href", function(d) { if (d.type == "author")
                                                      { return "graph-assets/user.png"}
                                                      else
                                                      { return "graph-assets/book.png"}})
                    .attr("x", node_image_offset)
                    .attr("y", node_image_offset)
                    .attr("width", node_image_size)        
                    .attr("height", node_image_size);
        
        5.  Author Hyperlinks
        
            The Author nodes are clickable, and link to a scholar search with the
            author's name.
            
                node.append("g")
                    .append("a")
                    .attr("xlink:href",function(d) {
                        if (d.type == "author")
                            return "http://scholar.google.com/scholar?q=" + encodeURIComponent(d.name)
                        else
                            return d.name+".pdf"})
                    .append("text")
                    .attr("dx", 12)
                    .attr("dy", 16)
                    .attr("text-anchor", "middle")
                    .text(function(d) { return d.name });
        
        6.  Initial Node Positions
        
            To help converging, the layout is initialized by setting all the nodes
            with attractor targets to their calculated target positions.
            
                nodes.forEach(function(node) {
                    if (node.x_target) node.x = node.x_target;
                    if (node.y_target) node.y = node.y_target;
                });
        
        7.  Initial Author Positions
        
            The initial positions of the author nodes are set to the positions of
            the paper nodes.  This is intended to allow the layout to converge
            faster, but does not work well.  When the layout starts, the first few
            cycles exhibit very high fluctuation amplitudes. (TODO: check if this
            is better after reordering)
            
                // source: author, target: paper
                links.forEach(function(link) {
                    var a_index = link.source;
                    var p_index = link.target;
                    nodes[a_index].x = nodes[p_index].x;
                    nodes[a_index].y = nodes[p_index].y;
                });
    
    4.  Force Layout Tick Handler<a id="orgtarget3"></a>
    
        This is the "hot loop" that actually updates all the svg elements
        according to the internal simulation.  It implements the attraction
        forces and updates the position of the svg nodes as well as their
        links.
        
            force.on("tick", function(e) {
                var kx = e.alpha * kx_mul;
                var ky = e.alpha * ky_mul;
            
                nodes.forEach(function(node) {
                    if (node.x_target)
                        node.x += (node.x_target - node.x) * kx;
                    if (node.y_target)
                        node.y += (node.y_target - node.y) * ky;
                });
            
                link.attr("x1", function(d) { return d.source.x; })
                    .attr("y1", function(d) { return d.source.y; })
                    .attr("x2", function(d) { return d.target.x; })
                    .attr("y2", function(d) { return d.target.y; });
            
                node.attr("transform", function(d) { return "translate(" + d.x + "," + d.y + ")"; });
            });

2.  Circle Packing Layout

    The circle packing layout is currently used for the classification
    view.
    
    The node value for this layout is a constant, resulting in
    evenly-sized leaf nodes (papers), which themselves are not actually
    displayed but only used as an attraction center point.
    (see [2.1](#orgheadline5))
    
        var pack = d3.layout.pack()
            .size([width , width])
            .value(function(d) { return 50; });
    
        function make_pack_layout() {
            <<pack-layout-initialization>>
        }
    
    1.  Pack Layout Initialization <a id="orgtarget4"></a>
    
        The Layout itself is created after data has been loaded by creating a
        svg group element for it (initially invisible).
        
            pack_svg = container.append("g")
                .attr("id", "pack_svg")
                .attr("opacity",0);
        
        `tnode` holds the actually created svg elements, using d3.js's enter
        selection mechanism.  If a node has no children, it is assigned the
        `leaf` class.  Also, the positions are already assigned here.  The
        actual representation is a `circle` element.
        
            var tnode = pack_svg.datum(tree).selectAll(".tnode")
                .data(pack.nodes)
                .enter().append("g")
                .attr("class", function(d) { return d.children ? "tnode" : "leaf tnode"; })
                .attr("transform", function(d) { return "translate(" + d.x + "," + d.y + ")"; });
            
            tnode.append("title")
                .attr("class", "pack")
                .text(function(d) {return d.name});
            
            tnode.append("circle")
                .attr("class", "pack")
                .attr("r", function(d) {return d.r});
        
        Labels for the categories are created, and moved a bit up from the
        center to increase readability.  The name is be clipped if it is too
        long.
        
            tnode.filter(function(d) { return d.children; }).append("g")
                .attr("transform", function(d) { return "translate(0," + (-d.r/10) + ")scale(" + Math.sqrt(d.r/50) + ")";})
                .append("text")
                .attr("class", "pack")
                .style("text-anchor", "middle")
                .text(function(d) { return d.name.substring(0, d.r / 3); });

### View Modes<a id="orgheadline31"></a>

The different layout modes are switched using `change_mode`, which
takes a mode string as a single argument.  Depending on the mode,
different parameters are used for the layouts.  In the end, opacities
are adjusted according to the mode, and the force layout is restarted
with the changed parameters.

    function change_mode(mode) {
        var pack_opacity, new_alpha, collab_opacity, link_strength, node_charge_mul;
    
        switch(mode) {
        case 'collaboration':
            <<collaboration-mode-parameters>>
            break;
        case 'category':
            <<classification-mode-parameters>>
            break;
        }
        d3.select("#pack_svg").attr("opacity", pack_opacity);
        container.selectAll(".link").attr("opacity", collab_opacity);
        container.selectAll(".node").filter(function(d) {return d.type == "author"}).attr("opacity", collab_opacity);
        /*force.charge(function(d) { return ((1-i) * node_charge(d))})*/
    
        force.charge(function(d) { return collab_charge(d) * node_charge_mul })
            .linkStrength(link_strength)
            .start()
            .alpha(new_alpha);
    }

1.  Collaboration Graph

    For the Collaboration Graph
    
    -   all paper nodes are attracted towards an individual point determined
        by [41](#orgsrcblock2)
    -   the horizontal force towards this target is lower then the vertical
        force
    -   the classification layout is hidden
    -   link strength is reduced to allow better clustering with papers as
        centers
    
        kx = 0.15;
        ky = 0.4;
        node_charge_mul = 1;
        new_alpha = 1;
        pack_opacity = 0;
        collab_opacity = 1;
        link_strength = 0.5;
        /* set the target coordinates for the papers*/
        nodes.forEach(function(node) {
            set_collab_paper_targets(node);
        });
    
    The attractor positions of the papers are a virtual grid, where the
    papers are ordered in x-direction by the first letter of the bibtex
    key, and in y-direction by the year of publication.  The y positions
    are compressed in a way that recent publications are spaced wider than
    older publications.
    
        function set_collab_paper_targets(node) {
            if (node.type == "paper") {
                // node.y_target = (((2016 - node.year))*20) + 200;
                node.y_target = (Math.sqrt(2016 - node.year) * 100) + 200;
                xmin = "A".charCodeAt(0);
                xmax = "Z".charCodeAt(0);
                xnode = node.name.toUpperCase().charCodeAt(0);
                node.x_target = Math.max(((xnode - xmin) / (xmax - xmin)) * width, 1);
            }
        }

2.  Classification Layout

    For the classification layout
    
    -   attractor force is the same for x and y
    -   node charge and link strength are zeroed to allow exact paper
        positioning
    -   the authors and links are made invisible, because they just flood
        the layout
    -   the attraction point for the paper nodes are set to the circle
        packing layout positions using [43](#orgsrcblock3)
    
        kx = 1;
        ky = 1;
        node_charge_mul = 0;
        new_alpha = 0.1;
        pack_opacity = 1;
        collab_opacity = 0;
        link_strength = 0;
        /* set the target coordinates for the papers*/
        nodes.forEach(function(node) {
            set_category_paper_targets(node);
        });
    
    The attractor positions are simply the centers of the calculated classification layout:
    
        function set_category_paper_targets(node) {
            node.x_target = node.pack_node.x;
            node.y_target = node.pack_node.y;
        }

### Application Initialization/Data Loading<a id="orgheadline32"></a>

Since we are using d3.js's json load function, everything that needs
to happen after loading must be clumsily put into the event handler to
that function.

This helper iterates through all the nodes in the `tree` data member
and creates links to the flat listed nodes.

    function connect_node(pnode) {
        if (pnode.children) pnode.children.forEach(connect_node);
        else {
            var fnode = nodes.find(function(d) {
                return d.name == pnode.name
            });
            if (fnode) {
                pnode.force_node = fnode;
                fnode.pack_node = pnode;
            }
        }
    }

After loading, the [data globals](#orgsrcblock4) are actually assigned the correct
values.  [14](#orgsrcblock5) is responsible for actually creating all layouts.

    d3.json("graph.json", function(error, json) {
        if (error) throw error;
    
        nodes = json.nodes;
        links = json.links;
        tree = json.tree;
    
        connect_node(tree);
    
        make_layout();
    });

# Hacking<a id="orgheadline35"></a>

This file is used to generate code and documentation.  It requires
org-mode which is supplied by emacs.  To (re-)generate the code file,
open this document and evaluate `org-babel-tangle`.
