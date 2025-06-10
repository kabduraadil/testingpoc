# testingpoc
testingpoc to generate text
pip install pandas openpyxl google-cloud-aiplatform
import streamlit as st
import json
from vertexai.generative_models import GenerativeModel

st.title("English to AWS Deequ Rule Generator")

# Initialize model once
@st.cache_resource
def get_model():
    return GenerativeModel(model_name="gemini-1.5-flash-preview-0409")

model = get_model()

# Advanced prompt template with placeholders
ADVANCED_PROMPT = """
You are a data quality expert converting English data quality rules into AWS Deequ JSON validation expressions.

Below are examples:

Attribute: cr_line_chng_typ_cd
English Rule: Produce an error if null or blank (Pre-cond: Acct_st_cl_moend is not 0)
Comment Tag: Dhub_dqr|cr_line_chng_typ_cd
Expected Output:
{{
  "comment": "Dhub_dqr|cr_line_chng_typ_cd",
  "name": "satisfies",
  "args": {{
    "column": "cr_line_chng_typ_cd IS NOT NULL and cr_line_chng_typ_cd != '' and Acct_st_cl_moend != 0",
    "condition": "==1.0"
  }}
}}

---

Attribute: {attribute}
English Rule: {rule}
Comment Tag: {comment}

Expected Output:
"""

def generate_rule(attribute: str, rule: str, comment: str):
    prompt = ADVANCED_PROMPT.format(attribute=attribute, rule=rule, comment=comment)
    response = model.generate_content(prompt, temperature=0.2, max_output_tokens=512)
    return response.text

# User inputs
attribute = st.text_input("Attribute name")
english_rule = st.text_area("English Rule")
comment_tag = st.text_input("Comment tag")

if st.button("Generate AWS Deequ Rule"):
    if not attribute or not english_rule or not comment_tag:
        st.warning("Please fill all fields.")
    else:
        with st.spinner("Generating..."):
            output = generate_rule(attribute, english_rule, comment_tag)
            st.subheader("Generated AWS Deequ JSON")
            try:
                json_output = json.loads(output)
                st.json(json_output)
            except Exception:
                st.text_area("Raw output (not valid JSON)", value=output, height=200)



import unittest
import json

def is_valid_deequ_rule(dq_rule_json):
    # Basic sanity checks for the generated DQ JSON structure
    required_keys = {"comment", "name", "args"}
    if not isinstance(dq_rule_json, dict):
        return False
    if not required_keys.issubset(dq_rule_json.keys()):
        return False
    if not isinstance(dq_rule_json.get("args"), dict):
        return False
    if "column" not in dq_rule_json["args"] or "condition" not in dq_rule_json["args"]:
        return False
    return True

class TestDeequRuleGeneration(unittest.TestCase):

    def test_sample_rule(self):
        sample_output = '''
        {
          "comment": "Dhub_dqr|cr_line_chng_typ_cd",
          "name": "satisfies",
          "args": {
            "column": "cr_line_chng_typ_cd IS NOT NULL and cr_line_chng_typ_cd != '' and Acct_st_cl_moend != 0",
            "condition": "==1.0"
          }
        }
        '''
        dq_rule_json = json.loads(sample_output)
        self.assertTrue(is_valid_deequ_rule(dq_rule_json))

    def test_invalid_rule(self):
        invalid_output = '{"comment": "missing args"}'
        dq_rule_json = json.loads(invalid_output)
        self.assertFalse(is_valid_deequ_rule(dq_rule_json))

if __name__ == "__main__":
    unittest.main()
----------

You are a data quality specialist who translates English data quality requirements into AWS Deequ validation JSON expressions.

Follow these rules:
- Include all relevant preconditions in the "column" expression.
- Use standard SQL syntax for column conditions.
- The "condition" field should be a string expression for the expected boolean condition.
- Always include the comment tag in the "comment" field.

### Examples:

Attribute: cr_line_chng_typ_cd
English Rule: Produce an error if null or blank (Pre-cond: Acct_st_cl_moend is not 0)
Comment Tag: Dhub_dqr|cr_line_chng_typ_cd

Expected Output:
{
  "comment": "Dhub_dqr|cr_line_chng_typ_cd",
  "name": "satisfies",
  "args": {
    "column": "cr_line_chng_typ_cd IS NOT NULL and cr_line_chng_typ_cd != '' and Acct_st_cl_moend != 0",
    "condition": "==1.0"
  }
}

---

Attribute: region_code
English Rule: Must not be in ('XX', 'YY', 'ZZ')
Comment Tag: Dhub_dqr|region_code

Expected Output:
{
  "comment": "Dhub_dqr|region_code",
  "name": "isContainedIn",
  "args": {
    "column": "region_code",
    "values": ["XX", "YY", "ZZ"]
  }
}

---

Attribute: customer_id
English Rule: Must be unique and not null
Comment Tag: Dhub_dqr|customer_id

Expected Output:
{
  "comment": "Dhub_dqr|customer_id",
  "name": "isUnique",
  "args": {
    "column": "customer_id"
  }
}

---

Attribute: transaction_date
English Rule: Must be a valid date in the past
Comment Tag: Dhub_dqr|transaction_date

Expected Output:
{
  "comment": "Dhub_dqr|transaction_date",
  "name": "satisfies",
  "args": {
    "column": "transaction_date < current_date",
    "condition": "==1.0"
  }
}

---

Attribute: {attribute_name}
English Rule: {english_rule}
Comment Tag: {comment_tag}

Expected Output:
