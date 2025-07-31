## **Optimized Plan** to generate step DAG


**Deprecation**
This RFC has been deprecated.


1. **User records a series of screenshots**  
   - Extract UI metadata (HTML structure, buttons, menus, etc.). 
   - Store timestamp & navigation context etc.

2. **Use LLMs to analyze intent of each screenshot**  
   - Send step screenshot, ui hierarchy structure, action element
   - Return:
     - **Primary Intent** (e.g., "Booking a Desk")  
     - **Action Taken** (e.g., "Clicked Book Now")  
     - **Possible Next Actions** (e.g., "Modify Reservation", "Cancel Booking")  
   - Compute **confidence score** for accuracy validation.

3. **Connect step with existing DAG**  
   - Match **action & next actions** with existing graph nodes using:
     - **Intent similarity**  
     - **UI structure comparison**  
     - **Confidence-based ranking**  
   - If ambiguity is **high**, queue step for **human review**.  
   - Prevent DAG fragmentation by:
     - **Merging similar nodes**
     - **Removing duplicate steps**

4. **Generate topic hierarchy based on the DAG**  
   - Create **dynamic views**:
     - **Task-based hierarchy** (e.g., “Booking a Desk”)  
     - **Feature-based hierarchy** (e.g., “Reservations”)  
     - **User-prioritized view** (based on workflow frequency)  
   - Apply **graph traversal algorithms** to create **optimized documentation flow**.

---

## **Benefits of the Optimization**
✔ **More Accurate Step Recognition** → By combining **screenshot + UI metadata + OCR**  
✔ **More Reliable DAG Matching** → Uses **confidence scoring & deduplication**  
✔ **Smarter Topic Hierarchy** → Supports **multiple hierarchical views**  
✔ **Handles Edge Cases Gracefully** → Uses **fallbacks & manual review when needed**  

