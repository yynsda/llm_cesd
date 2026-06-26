import streamlit as st
import json
import os
from openai import OpenAI
st.set_page_config(
    page_title="CES-D Profile Assessment",
    page_icon="🧠",
    layout="centered"
)
DISCLAIMER="""
⚠️ **Disclaimer**
This tool is for scientific research purposes only and does not constitute medical diagnosis or treatment advice.
The assessment results cannot replace evaluation by a qualified psychiatrist, psychologist, or other licensed mental health professional.
If you or a family member is experiencing mental health difficulties, please seek professional medical help in a timely manner.
"""
CESD_ITEMS=[
    ("item1","During the past week, were you bothered by things that usually do not bother you?","Bothered by little things",False),
    ("item2","During the past week, did you have trouble keeping your mind on what you were doing?","Difficulty concentrating",False),
    ("item3","During the past week, did you feel depressed or low in spirits?","Depressed mood",False),
    ("item4","During the past week, did you feel that everything you did was an effort?","Feeling that everything was an effort",False),
    ("item5","During the past week, did you feel hopeful about the future?","Feeling hopeful about the future",True),
    ("item6","During the past week, did you feel fearful?","Feeling fearful",False),
    ("item7","During the past week, was your sleep restless?","Restless sleep",False),
    ("item8","During the past week, did you feel happy?","Feeling happy",True),
    ("item9","During the past week, did you feel lonely?","Feeling lonely",False),
    ("item10","During the past week, did you feel that you could not get going?","Could not get going",False),
]
FREQ_OPTIONS={
    "Rarely or none of the time (less than 1 day)":0,
    "Some or a little of the time (1–2 days)":1,
    "Occasionally or a moderate amount of the time (3–4 days)":2,
    "Most or all of the time (5–7 days)":3,
}
PREFER_NOT="Prefer not to say"
YES="Yes"
NO="No"
@st.cache_resource
def load_api_key():
    try:
        return st.secrets["deepseek_api_key"]
    except Exception:
        return None
def generate_advice(prompt:str)->str:
    client=OpenAI(
        api_key=st.secrets["deepseek_api_key"],
        base_url="https://api.deepseek.com"
    )
    resp=client.chat.completions.create(
        model="deepseek-chat",
        temperature=0.2,
        max_tokens=700,
        messages=[{"role":"user","content":prompt}]
    )
    return resp.choices[0].message.content.strip()
def reverse_encode(is_reverse:bool,score:int)->int:
    return 3-score if is_reverse else score
def collect_user_context():
    st.header("👤 Individual Background Information")
    st.caption("The following information is used only to make the individualized advice more relevant to daily life. It is not used for symptom profiling.")
    ctx={}
    ctx["age"]=st.number_input("Age (years)",min_value=60,max_value=100,value=65,step=1)
    ctx["gender"]=st.selectbox("Gender",[PREFER_NOT,"Male","Female"])
    ctx["marital_status"]=st.selectbox("Marital status",[PREFER_NOT,"Married","Other (never married, divorced, or widowed)"])
    ctx["residence"]=st.selectbox("Place of residence",[PREFER_NOT,"Urban","Rural"])
    ctx["education"]=st.selectbox("Education level",[PREFER_NOT,"Below primary school","Primary school","Middle school","High school or above"])
    ctx["sleep_duration"]=st.number_input("Average nighttime sleep duration in the past week (hours)",min_value=0.0,max_value=12.0,value=6.0,step=0.5)
    ctx["pain"]=st.selectbox("Do you currently have pain in any part of your body?",[PREFER_NOT,NO,YES])
    ctx["fall"]=st.selectbox("Have you had at least one fall in the past year?",[PREFER_NOT,NO,YES])
    ctx["function_limitation"]=st.selectbox("Do you have limitations in daily physical functioning?",[PREFER_NOT,"No difficulty","Difficult but can still complete independently","Difficult and need help","Unable to complete"])
    ctx["chronic"]=st.selectbox("Do you have any chronic disease?",[PREFER_NOT,NO,YES])
    ctx["adl_continence"]=st.selectbox("Do you have difficulty controlling urination or bowel movements?",[PREFER_NOT,"No difficulty","Difficult but can still manage independently","Difficult and need help","Unable to manage"])
    ctx["physical_activity"]=st.selectbox("Physical activity level",[PREFER_NOT,"Sedentary","Low","Moderate","High"])
    ctx["social_activity"]=st.selectbox("Do you often participate in social activities, such as visiting others or community activities?",[PREFER_NOT,NO,YES])
    ctx["episodic_memory"]=st.number_input("Episodic memory score (0–10)",min_value=0,max_value=10,value=6,step=1)
    ctx["mental_status"]=st.number_input("Mental status score (0–11)",min_value=0,max_value=11,value=7,step=1)
    return ctx
def format_context_block(ctx:dict)->str:
    core_lines=[]
    pain_val=ctx.get("pain",PREFER_NOT)
    if pain_val!=PREFER_NOT:
        core_lines.append(f"Pain: {pain_val}")
    fl_val=ctx.get("function_limitation",PREFER_NOT)
    if fl_val!=PREFER_NOT:
        core_lines.append(f"Daily functional limitation: {fl_val}")
    fall_val=ctx.get("fall",PREFER_NOT)
    if fall_val!=PREFER_NOT:
        core_lines.append(f"Fall history: {fall_val}")
    sleep_val=ctx.get("sleep_duration")
    core_lines.append(f"Nighttime sleep duration: {sleep_val} hours")
    basic_lines=[]
    for k,label in [("gender","Gender"),("age","Age"),("marital_status","Marital status"),("residence","Residence"),("education","Education")]:
        v=ctx.get(k,PREFER_NOT)
        if v!=PREFER_NOT:
            unit=" years" if k=="age" else ""
            basic_lines.append(f"{label}: {v}{unit}")
    other_lines=[]
    for k,label in [("physical_activity","Physical activity"),("social_activity","Social activity"),("chronic","Chronic disease"),("adl_continence","Urinary or bowel control difficulty")]:
        v=ctx.get(k,PREFER_NOT)
        if v!=PREFER_NOT:
            other_lines.append(f"{label}: {v}")
    em=ctx.get("episodic_memory")
    ms=ctx.get("mental_status")
    other_lines.append(f"Episodic memory score: {em}/10 | Mental status score: {ms}/11")
    sections=[]
    if core_lines:
        sections.append("[Core Health Status]\n"+" | ".join(core_lines))
    if basic_lines:
        sections.append("[Basic Information]\n"+" | ".join(basic_lines))
    if other_lines:
        sections.append("[Other Background Information]\n"+" | ".join(other_lines))
    return "\n\n".join(sections)
def build_risk_block(ctx:dict,item_details:dict)->str:
    risk_rules=[]
    sleep_score=item_details.get("item7",("",0,"",False))[1]
    low_mood_score=item_details.get("item3",("",0,"",False))[1]
    hopeless_score=item_details.get("item5",("",0,"",False))[1]
    lonely_score=item_details.get("item9",("",0,"",False))[1]
    effort_score=item_details.get("item4",("",0,"",False))[1]
    fatigue_score=item_details.get("item10",("",0,"",False))[1]
    if ctx.get("mental_status",99)<=4 or ctx.get("episodic_memory",99)<=3:
        risk_rules.append("Lower cognitive or memory score: the advice must be simpler and should include family support for recording symptoms, reminding daily routines, or accompanying the person to medical visits. Do not use diagnostic terms such as dementia or cognitive impairment.")
    if ctx.get("fall")==YES:
        risk_rules.append("History of falls: if activity is recommended, safety must come first. Recommend only low-intensity activities such as slow indoor walking, seated stretching, or ankle movements. Avoid outdoor, high-intensity, or unsupervised activities.")
    if ctx.get("function_limitation") in ["Difficult and need help","Unable to complete"]:
        risk_rules.append("Marked functional limitation: recommend family support and do not ask the user to complete complex or outdoor activities alone.")
    elif ctx.get("function_limitation")=="Difficult but can still complete independently":
        risk_rules.append("Mild functional limitation: activity suggestions should be low-intensity, brief, and gradual.")
    if ctx.get("adl_continence") in ["Difficult but can still manage independently","Difficult and need help","Unable to manage"]:
        risk_rules.append("Difficulty controlling urination or bowel movements: protect privacy, suggest recording frequency and timing, and advise medical consultation if needed. Use a natural tone and avoid making the user feel ashamed.")
    if ctx.get("pain")==YES:
        risk_rules.append("Physical pain: if sleep problems, worries, low activity, or lack of energy are present, remind the user that pain may affect daily functioning and suggest medical evaluation for the cause of pain. Do not simply attribute these problems to psychological causes.")
    if ctx.get("sleep_duration",99)<6 or sleep_score>=2:
        risk_rules.append("Insufficient sleep or obvious restless sleep: prioritize sleep-rhythm advice, such as keeping a fixed wake-up time, relaxing before bed, and avoiding overly long naps.")
    if ctx.get("social_activity")==NO and (lonely_score>0 or low_mood_score>0 or hopeless_score>0):
        risk_rules.append("Limited social activity with loneliness, low mood, or hopelessness: suggest low-pressure contact first, such as calling children, relatives, or familiar neighbors. Do not directly require the user to join group activities.")
    if ctx.get("physical_activity") in ["Sedentary","Low"] and (effort_score>0 or fatigue_score>0 or low_mood_score>0):
        risk_rules.append("Low physical activity with effortfulness, lack of energy, or low mood: suggest brief, low-intensity activity. If there is also a fall history or functional limitation, emphasize safety and family supervision.")
    if ctx.get("chronic")==YES and (ctx.get("pain")==YES or sleep_score>=2 or ctx.get("function_limitation")!="No difficulty" or effort_score>0 or fatigue_score>0):
        risk_rules.append("Chronic disease with pain, sleep, activity, or functional problems: gently remind the user to follow existing medical advice and attend follow-up care, but do not infer that emotional difficulties are caused by chronic disease.")
    if ctx.get("residence")=="Rural" or ctx.get("education") in ["Below primary school","Primary school"]:
        risk_rules.append("Rural residence or lower education level: use plain language, keep suggestions close to daily life, and avoid complex psychological concepts.")
    if not risk_rules:
        return "No background risks require special emphasis."
    return "\n".join([f"{i+1}. {rule}" for i,rule in enumerate(risk_rules)])
def build_prompt(item_details:dict,cesd_total:int,ctx:dict)->str:
    item_lines=[]
    high_items=[]
    for key,(choice_label,encoded,zh_name,is_reverse) in item_details.items():
        note=" (reverse-scored; a higher score indicates greater symptom burden in this domain)" if is_reverse else ""
        item_lines.append(f"  {key} \"{zh_name}\"{note}: {choice_label} -> encoded score={encoded}")
        if encoded>0:
            high_items.append((key,zh_name,encoded,choice_label))
    item_block="\n".join(item_lines)
    high_items_sorted=sorted(high_items,key=lambda x:x[2],reverse=True)
    if high_items_sorted:
        high_block="\n".join([
            f"  {key} \"{zh_name}\": {choice_label} (score={encoded})"
            for key,zh_name,encoded,choice_label in high_items_sorted
        ])
    else:
        high_block="  No item has a score greater than 0; the overall status appears good."
    context_block=format_context_block(ctx)
    risk_block=build_risk_block(ctx,item_details)
    dep_status="positive (>=10 points)" if cesd_total>=10 else "negative (<10 points)"
    if cesd_total>=20:
        referral_rule="Total score >=20: clearly advise the user to seek professional assessment from a psychiatry, psychology, or geriatric clinic as soon as possible and not to delay."
    elif cesd_total>=10:
        referral_rule="Total score 10–19: suggest consulting a community clinic or mental health professional soon. The tone should be reassuring but should not minimize the concern."
    else:
        referral_rule="Total score <10: gently remind the user that if symptoms last longer than two weeks or clearly worsen, they may talk with a community doctor or mental health professional."
    prompt=f"""You are a clinical psychologist experienced in mental health care for older adults. You are providing individualized support advice for a Chinese older adult aged 60 years or above.
[Item-by-item responses] (total score: {cesd_total}/30; depressive symptom screening: {dep_status})
{item_block}
[Symptomatic items only: score > 0, ranked by severity]
{high_block}
{context_block}
[Background risks that should be prioritized]
{risk_block}
[Referral guidance rule]
{referral_rule}
[Your task]
Based on the information above, write individualized support advice in warm, plain English. Strictly follow these rules:
1. The first paragraph describing the person's current emotional state must be based only on the symptomatic items. Psychological items with a score of 0 must not be described as existing problems. Later suggestions may use key background risks to improve safety and feasibility.
2. If all items have a score of 0, state that the overall status appears good and provide only preventive suggestions. Do not exaggerate symptoms.
3. First paragraph (1–2 sentences): describe the person's current emotional state in simple language and mention only all of the highest-scoring items. Avoid professional jargon.
4. Second paragraph: provide 2–3 numbered suggestions. Each suggestion must directly correspond to one high-scoring symptom or one key background risk. Do not mechanically list all background variables. Integrate no more than 2–3 of the most relevant background factors.
5. Background information is used only to make the advice safer, more feasible, and closer to daily life. Do not write all background variables into the text.
6. Prioritize the content in the background risk section. If it includes lower cognitive or memory score, fall history, marked functional limitation, urinary or bowel control difficulty, pain, or obvious sleep insufficiency, include at least 1–2 of these issues in the advice.
7. Gender, age, marital status, residence, and education are generally not written directly in the text. They should only be used to adjust language style and difficulty, avoiding stereotyped wording.
8. If the user chooses \"Prefer not to say\", never describe that information as a definite fact.
9. The final sentence must provide referral advice according to the referral guidance rule.
10. Do not use professional terms such as template, profile, CES-D, scale, or encoded score in the final advice.
11. Do not provide medication advice, do not make a diagnosis, and do not promise treatment effects.
12. Use a warm tone, like a family doctor speaking gently with an older adult.
Output only the advice text. Do not add any title or label."""
    return prompt
def generate_advice(api_key:str,prompt:str)->str:
    client=OpenAI(api_key=api_key,base_url="https://api.deepseek.com")
    resp=client.chat.completions.create(
        model="deepseek-chat",
        temperature=0.2,
        max_tokens=700,
        messages=[{"role":"user","content":prompt}]
    )
    return resp.choices[0].message.content.strip()
def main():
    st.title("🧠 CES-D Mental Health Support Advice Generator")
    st.markdown("**Individualized Depressive Symptom Support Assessment**")
    st.info(DISCLAIMER)
    st.divider()
    api_key=load_api_key()
    if not api_key:
        st.warning("⚠️ DeepSeek API Key was not found. Please check your Streamlit Secrets.")
    st.caption("Please choose the option that best describes how you felt during the **past week**.")
    encoded_scores={}
    item_details={}
    for key,q_en,en_name,is_reverse in CESD_ITEMS:
        if is_reverse:
            st.caption("🔄 This item is reverse-scored: choosing 'Rarely or none of the time' indicates a higher symptom burden in this domain.")
        choice=st.radio(q_en,options=list(FREQ_OPTIONS.keys()),key=key,horizontal=False)
        raw=FREQ_OPTIONS[choice]
        encoded=reverse_encode(is_reverse,raw)
        encoded_scores[key]=encoded
        item_details[key]=(choice,encoded,en_name,is_reverse)
    st.divider()
    user_context=collect_user_context()
    st.divider()
    cesd_total=sum(encoded_scores.values())
    dep_positive=cesd_total>=10
    col1,col2=st.columns(2)
    with col1:
        st.metric("Total score",f"{cesd_total} / 30")
    with col2:
        if dep_positive:
            st.metric("Depressive symptom screening","Positive",delta=">=10 points",delta_color="inverse")
        else:
            st.metric("Depressive symptom screening","Negative",delta="<10 points",delta_color="normal")
    if st.button("💬 Generate individualized support advice",type="primary",use_container_width=True):
        st.divider()
        st.header("💬 Individualized Support Advice")
        if not api_key:
            st.warning(f"DeepSeek API key was not found, so advice cannot be generated. Please check: {config_path}")
        else:
            prompt=build_prompt(item_details,cesd_total,user_context)
            with st.spinner("Generating individualized advice..."):
                try:
                    advice=generate_advice(prompt)
                    st.markdown(advice)
                except Exception as e:
                    st.error(f"Generation failed: {e}")
        st.divider()
        st.caption("⚠️ This assessment is for reference only and does not constitute a medical diagnosis. If professional help is needed, please contact a psychiatrist, psychologist, or other qualified mental health professional.")
if __name__=="__main__":
    main()
