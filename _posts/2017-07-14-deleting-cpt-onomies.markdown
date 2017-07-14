---
layout: post
title:  "Getting Rid of CPT-onomies and Switching to Custom Fields"
date:   2017-07-14 11:10:03 -0700
categories: wordpress cptonomies update
---

After months of frustration with the CPT-onomies Wordpress plugin, including
memory leaks, conflicts with caching plugins, broken links, incredibly slow load
time and more, I decided it was time to get rid of the plugin.

![CPT-onomies deleted]({{ site.url }}/assets/cpt-onomies-deleted.png)

The Wordpress site I was working on used CPT-onomies to create a custom Issue
CPT-onomy and it had over 2,000 articles which were each assigned an Issue.
In lieu of allowing CPT-onomies to create the Issue CPT-onomy, I decided to register a
regular post type with the same slug. To register the post type, I used
[this site](https://generatewp.com/post-type/) to generate the desired configuration
(similar to how I would have changed post type settings in CPT-onomies) and
then added the code it produced to functions.php in the main Wordpress theme directory.

{% highlight php %}
/* Register Custom Post Type */
function issue_generator() {

	$labels = array(
		'name'                  => _x( 'Issues', 'Post Type General Name', 'text_domain' ),
		'singular_name'         => _x( 'Issue', 'Post Type Singular Name', 'text_domain' ),
		'menu_name'             => __( 'Issues', 'text_domain' ),
		'name_admin_bar'        => __( 'Issues', 'text_domain' ),
		'archives'              => __( 'Item Archives', 'text_domain' ),
		'attributes'            => __( 'Item Attributes', 'text_domain' ),
		'parent_item_colon'     => __( 'Parent Item:', 'text_domain' ),
		'all_items'             => __( 'All Items', 'text_domain' ),
		'add_new_item'          => __( 'Add New Item', 'text_domain' ),
		'add_new'               => __( 'Add New', 'text_domain' ),
		'new_item'              => __( 'New Item', 'text_domain' ),
		'edit_item'             => __( 'Edit Item', 'text_domain' ),
		'update_item'           => __( 'Update Item', 'text_domain' ),
		'view_item'             => __( 'View Item', 'text_domain' ),
		'view_items'            => __( 'View Items', 'text_domain' ),
		'search_items'          => __( 'Search Item', 'text_domain' ),
		'not_found'             => __( 'Not found', 'text_domain' ),
		'not_found_in_trash'    => __( 'Not found in Trash', 'text_domain' ),
		'featured_image'        => __( 'Featured Image', 'text_domain' ),
		'set_featured_image'    => __( 'Set featured image', 'text_domain' ),
		'remove_featured_image' => __( 'Remove featured image', 'text_domain' ),
		'use_featured_image'    => __( 'Use as featured image', 'text_domain' ),
		'insert_into_item'      => __( 'Insert into item', 'text_domain' ),
		'uploaded_to_this_item' => __( 'Uploaded to this item', 'text_domain' ),
		'items_list'            => __( 'Items list', 'text_domain' ),
		'items_list_navigation' => __( 'Items list navigation', 'text_domain' ),
		'filter_items_list'     => __( 'Filter items list', 'text_domain' ),
	);
	$rewrite = array(
		'slug'                  => 'issue',
		'with_front'            => true,
		'pages'                 => true,
		'feeds'                 => true,
	);
	$args = array(
		'label'                 => __( 'Issue', 'text_domain' ),
		'description'           => __( 'The issue post type', 'text_domain' ),
		'labels'                => $labels,
		'supports'              => array( 'title', 'excerpt', 'thumbnail', 'page-attributes', ),
		'hierarchical'          => false,
		'public'                => true,
		'show_ui'               => true,
		'show_in_menu'          => true,
		'menu_position'         => 5,
		'show_in_admin_bar'     => true,
		'show_in_nav_menus'     => true,
		'can_export'            => true,
		'has_archive'           => 'issue/tax/$term_slug',
		'exclude_from_search'   => false,
		'publicly_queryable'    => true,
		'rewrite'               => $rewrite,
		'capability_type'       => 'page',
    'show_ui'                 => true,
    'show_in_rest'            => true,
    'rest_base'              => 'foo',
    'rest_controller_class'   => 'WP_REST_Posts_Controller',
	);
	register_post_type( 'issue', $args );

}
add_action( 'init', 'issue_generator', 0 );
{% endhighlight %}

And, after deactivating CPT-onomies, my site still worked! All of my old Issue posts appeared
as a regular custom post type rather than as a CPT-onomy. Furthermore, on the single
Issue pages the children of the Issue post that I had assigned using CPT-onomies
still appeared. Why? Well, CPT-onomies was simply using regular post meta data
using the meta key '_custom_post_type_onomies_relationship'. In content-issue.php,
I was using the following loop to display my posts:

{% highlight php %}
$query = new WP_Query(array(
'post_type' => 'post',
'posts_per_page' => '100',
'meta_key' => '_custom_post_type_onomies_relationship',  
'meta_value' => array($issue->ID)));

{% endhighlight %}

... So when I deactivated and deleted CPT-onomies, the meta data was still in the database
under the same key without all of the overhead that CPT-onomies created.

Next, I had to figure out how to update the meta data for the key '_custom_post_type_onomies_relationship'
from the Wordpress admin interface. I wanted a column for Issue titles to be displayed on the admin posts page
so I added the following code to functions.php:

{% highlight php %}
add_action("manage_posts_custom_column", "issue_custom_columns");

function issue_custom_columns($column){
	global $post;
	if( $column == 'issue' ) {
			$issue_id = get_post_meta( $post->ID, '_custom_post_type_onomies_relationship', true );
		 if (!empty($issue_id)) {
			$issue_title = get_the_title($issue_id);
			echo $issue_title;
		}
	}
}

add_filter("manage_edit-post_columns", "add_issue_columns");


function add_issue_columns($columns){
	    $columns['issue'] = 'Issue';
	    return $columns;

}
{% endhighlight %}

In addition to the Issue column on the posts page, I wanted to be able to be able to change
to Issue field on each individual post during editing. For this, I needed a custom
metabox that allowed me to select an Issue and save the Issue ID in the field
'_custom_post_type_onomies_relationship'. I decided to use Advanced Custom Fields (ACF),
a popular and lightweight Wordpress plugin and create a metabox with a [post object
field](https://www.advancedcustomfields.com/resources/post-object/).

![CPT-onomies ACF Field]({{ site.url }}/assets/acf-cpt-onomies-field.png)

In ACF, I set "select multiple values" to "no" (this is crucial so that post object only returns
  one ID to the meta field like the CPT-onomies plugin did). Now, whenever I create a new post
  I see this metabox which allows me to select an Issue.

![CPT-onomies ACF Post Object Metabox]({{ site.url }}/assets/acf-issue-metabox.png)

I tested out created a new Issue and also creating a new post and assigning it the new Issue -
and it worked! No broken links, no unpredictable site lag, it just worked.
