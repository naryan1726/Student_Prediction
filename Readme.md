## End to End Machine learning Project 


import pandas as pd
import re

def clean_dataframes(dataframes_dict):
    """
    Clean extracted dataframes by pulling out relevant data
    
    Args:
        dataframes_dict: Dictionary with dataframe names as keys and dataframes as values
        
    Returns:
        Dictionary with "[original_name]_cleaned" as keys and cleaned dataframes as values
    """
    cleaned_dataframes = {}
    
    for name, df in dataframes_dict.items():
        print(f"Cleaning dataframe: {name}")
        
        # Extract title from rows 2-5
        title = extract_title_from_df(df)
        
        # Find columns for Grade, Job Code, Title if they exist
        columns_info = identify_key_columns(df)
        
        # Find Annual rows and extract start/end rates
        annual_data = extract_annual_data(df, columns_info)
        
        # Convert the list of dictionaries to a DataFrame and add category column
        if annual_data:
            result_df = pd.DataFrame(annual_data)
            # Add the title as a category column
            result_df['category'] = title
            
            # Use original name + "_cleaned" as the key
            cleaned_name = f"{name}_cleaned"
            cleaned_dataframes[cleaned_name] = result_df
            print(f"  Successfully cleaned data for: {name}")
        else:
            print(f"  No suitable data found in: {name}")
    
    return cleaned_dataframes

def extract_title_from_df(df):
    """Extract title from rows 2-5 of dataframe"""
    title = ""
    
    # Convert dataframe index to ensure we can access by integer position
    df_copy = df.reset_index(drop=True) if not isinstance(df.index, pd.RangeIndex) else df.copy()
    
    # Look through rows 2-5 (index 1-4) for title
    for i in range(1, 5):
        if i >= len(df_copy):
            break
            
        row = df_copy.iloc[i]
        # Filter out NaN and empty strings, join remaining strings
        title_parts = [str(val).strip() for val in row if pd.notna(val) and str(val).strip() and str(val).upper() != 'NAN']
        
        if title_parts:
            if 'SCHEDULE' in ' '.join(title_parts) or 'BUREAU' in ' '.join(title_parts):
                title += ' '.join(title_parts) + ' '
    
    return title.strip()

def identify_key_columns(df):
    """Find columns for Grade, Job Code, Title"""
    columns_info = {
        'grade_col': None,
        'job_code_col': None,
        'title_col': None,
        'header_row': None
    }
    
    # First, find the header row containing 'Grade' or other key columns
    for i, row in df.iterrows():
        row_values = [str(val).lower() if pd.notna(val) else "" for val in row]
        row_text = " ".join(row_values)
        
        if "grade" in row_text:
            columns_info['header_row'] = i
            # Now identify specific columns in this row
            for j, val in enumerate(row):
                if pd.notna(val):
                    val_str = str(val).lower()
                    if "grade" in val_str:
                        columns_info['grade_col'] = j
                    elif "job" in val_str and "code" in val_str:
                        columns_info['job_code_col'] = j
                    elif "title" in val_str:
                        columns_info['title_col'] = j
            break
    
    return columns_info

def extract_annual_data(df, columns_info):
    """Extract annual data with start and end rates"""
    annual_data = []
    
    # If we found a header row
    if columns_info['header_row'] is not None:
        # Create a new dataframe with proper headers
        header_row = columns_info['header_row']
        df_with_headers = df.iloc[header_row+1:].copy()
        df_with_headers.columns = df.iloc[header_row]
        
        # Find annual rows
        annual_rows = []
        for idx, row in df_with_headers.iterrows():
            for val in row:
                if isinstance(val, str) and "annual" in val.lower():
                    annual_rows.append(idx)
                    break
        
        # Find start rate column (Entry Rate or 1st Step)
        start_rate_col = None
        for col in df_with_headers.columns:
            if pd.notna(col) and isinstance(col, str):
                if "entry rate" in str(col).lower() or "1st step" in str(col).lower():
                    start_rate_col = col
                    break
        
        # Find end rate column (highest step)
        end_rate_col = None
        step_cols = {}
        for col in df_with_headers.columns:
            if pd.notna(col) and isinstance(col, str):
                step_match = re.search(r'(\d+)(th|st|nd|rd)\s+step', str(col).lower())
                if step_match:
                    step_num = int(step_match.group(1))
                    step_cols[step_num] = col
        
        if step_cols:
            highest_step = max(step_cols.keys())
            end_rate_col = step_cols[highest_step]
        
        # Extract data for each annual row
        if annual_rows and start_rate_col and end_rate_col:
            for idx in annual_rows:
                row_data = df_with_headers.loc[idx]
                
                data_item = {
                    'Grade': row_data[columns_info['grade_col']] if columns_info['grade_col'] is not None else None,
                    'Term': 'Annual',
                    'Start_Rate': row_data[start_rate_col],
                    'End_Rate': row_data[end_rate_col]
                }
                
                # Add job code and title if available
                if columns_info['job_code_col'] is not None:
                    data_item['Job_Code'] = row_data[columns_info['job_code_col']]
                if columns_info['title_col'] is not None:
                    data_item['Title'] = row_data[columns_info['title_col']]
                
                annual_data.append(data_item)
    
    return annual_data

# Example usage:
# Assuming you have a dictionary of already extracted dataframes
# extracted_dataframes = {'df1': df1, 'df2': df2, ...}
# cleaned_dataframes = clean_dataframes(extracted_dataframes)
