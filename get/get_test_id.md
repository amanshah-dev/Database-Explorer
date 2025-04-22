CREATE FUNCTION get_test_id(subjectname text) RETURNS bigint
    LANGUAGE plpgsql
AS
$$
DECLARE
    testId BIGINT;
BEGIN
    SELECT CASE subjectName
               WHEN 'Anatomy' THEN 2123625
               WHEN 'Anesthesia' THEN 2123642
               WHEN 'Biochemistry' THEN 2123627
               WHEN 'Dermatology' THEN 2123639
               WHEN 'ENT' THEN 2123632
               WHEN 'Forensic Medicine and Toxicology' THEN 2123631
               WHEN 'Medicine' THEN 2123635
               WHEN 'Microbiology' THEN 2123629
               WHEN 'Obs - Gynae' THEN 2123637
               WHEN 'Ophthalmology' THEN 2123633
               WHEN 'Orthopedics' THEN 2123641
               WHEN 'PSM' THEN 2123634
               WHEN 'Pathology' THEN 2123628
               WHEN 'Pediatrics' THEN 2123638
               WHEN 'Pharmacology' THEN 2123630
               WHEN 'Physiology' THEN 2123626
               WHEN 'Psychiatry' THEN 2123640
               WHEN 'Radiology' THEN 2123643
               WHEN 'Retina' THEN 2630778
               WHEN 'Surgery' THEN 2123636
               WHEN 'Synapse I' THEN 2627044
               WHEN 'Synapse II' THEN 2627057
               WHEN 'Synapse III' THEN 2627058
               END
    INTO testId;

    RETURN testId;
END;
$$;

ALTER FUNCTION get_test_id(text) OWNER TO learner;
