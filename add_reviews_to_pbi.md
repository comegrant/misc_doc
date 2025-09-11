# Adding Comments & Quick Comments to Power BI

## Decision points

- We can have either 1 `dim_recipe_reviews` containing both comments & ratings. Or two separate tables. I like option 1 for the sake of keeping the schema simple.

- Do we want to support anonymous comments? If so, we need to use the native `pim.dbo.recipes_rating.recipe_rating_id` instead of `concat(billing_agreement_id,recipe_id)`. 

- Do we keep duplicates columns in `fact_orders` and `dim_recipe_reviews` (e.g: `recipe_rating`)? This might cause confusion if we decide to support anonymous reviews. Since ratings & comments without an `agreement_billing_id` cannot be in `fact_orders`.

- We can either include `quick_comment_combination` in the regular `recipe_comment` column along with a boolean flag `is_quick_comment` or keep them in separate columns. I like the former since it would be easier for an end user in PBI to filter out quick comments than to coalesce regular comments and quick comments.

## Proposed schema:

```mermaid
erDiagram
direction TB
FACT_ORDERS ||--o| DIM_RECIPE_REVIEWS: can_have
FACT_ORDERS ||--o{ BRIDGE_RECIPE_QUICK_COMMENTS: can_have
BRIDGE_RECIPE_QUICK_COMMENTS }|--|| DIM_QUICK_COMMENTS: has_many


FACT_ORDERS {
    string fk_dim_recipe_reviews FK "added (hash of recipe_rating_id)"
    string fk_bridge_recipes_quick_comments FK "added (hash of recipe_rating_id for quick comments only)"
    string recipe_rating_id "switch to native recipe_rating_id"
    string recipe_comment_id "remove, redundant with recipe_rating_id"
    int recipe_rating "remove?"
    int recipe_rating_score "remove?"
    string recipe_comment "remove"
}
DIM_RECIPE_REVIEWS {
    string pk_dim_recipe_reviews PK
    string fk_dim_recipes FK "required to support anonymous reviews"
    date recipe_review_created
    timestamp recipe_review_created_at
    string recipe_rating_id
    string recipe_comment_id "remove, redundant with recipe_rating_id"
    int recipe_rating
    int recipe_rating_score
    string recipe_comment
    string all_quick_comments_local
    string all_quick_comments_english
    int number_of_quick_comments
    bool is_quick_comments
    bool is_anonymous_review
    bool is_not_cooked_dish
}
BRIDGE_RECIPE_QUICK_COMMENTS {
    string pk_bridge_quick_comments PK
    string fk_dim_quick_comments FK
}
DIM_QUICK_COMMENTS {
    string pk_dim_quick_comments PK
    string quick_comment_id 
    string company_id
    int quick_comment_language_id
    string quick_comment_language_name
    string quick_comment_text_local
    string quick_comment_text_english
}
```
