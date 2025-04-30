import requests
import openai
import json
from flask import Flask, request, jsonify

app = Flask(__name__)

# ✅ API Keys
ZILLOW_API_KEY = "7669d41fafmsh726958cb0735ff5p18abf3jsnc86cfe210aac"
ZILLOW_API_HOST = "zillow-com1.p.rapidapi.com"
openai_api_key = os.getenv("OPENAI_API_KEY")


HEADERS = {
    "X-RapidAPI-Key": ZILLOW_API_KEY,
    "X-RapidAPI-Host": ZILLOW_API_HOST
}

openai.api_key = OPENAI_API_KEY

@app.route("/analyze-property", methods=["POST"])
def analyze_property():
    data = request.get_json()
    address = data.get("address")
    zipcode = data.get("zip")

    if not address or not zipcode:
        return jsonify({"error": "Address and zip are required"}), 400

    # --- STEP 1: GET ZPID ---
    prop_res = requests.get(
        "https://zillow-com1.p.rapidapi.com/property",
        headers=HEADERS,
        params={"address": address, "zipcode": zipcode}
    )
    prop_data = prop_res.json()
    zpid = prop_data.get("zpid")
    if not zpid:
        return jsonify({"error": "ZPID not found"}), 404

    # --- STEP 2: GET Additional Property Data ---
    comps = requests.get("https://zillow-com1.p.rapidapi.com/propertyComps", headers=HEADERS, params={"zpid": zpid}).json()
    tax_history = prop_data.get("taxHistory", [])
    latest_tax = tax_history[0].get("taxPaid") if tax_history else None

    rent_zestimate = prop_data.get("rentZestimate")
    zestimate = prop_data.get("zestimate")
    sqft = (
        prop_data.get("livingArea")
        or prop_data.get("livingAreaValue")
        or prop_data.get("livingAreaRange")
        or comps.get("livingArea")
    )
    year_built = prop_data.get("yearBuilt")
    photo = prop_data.get("imgSrc")
    price = prop_data.get("price") or comps.get("price")

    full_info = {
        "address": address,
        "price": price,
        "beds": prop_data.get("bedrooms"),
        "baths": prop_data.get("bathrooms"),
        "sqft": sqft,
        "tax": latest_tax,
        "yearBuilt": year_built,
        "zestimate": zestimate,
        "rentZestimate": rent_zestimate,
        "photo": photo,
        "description": prop_data.get("description")
    }

    prompt = f"""Analyze the following property for real estate investment. Provide ROI estimate, rent/price insight, tax burden, year-built risks, and anything insightful for an investor:

{json.dumps(full_info, indent=2)}"""

    chat = openai.chat.completions.create(
        model="gpt-4",
        messages=[
            {"role": "system", "content": "You are a real estate investment analyst."},
            {"role": "user", "content": prompt}
        ],
        temperature=0.7
    )

    return jsonify({
        "photo": photo,
        "summary": chat.choices[0].message.content.strip()
    })

if __name__ == "__main__":
    app.run(debug=True)
