create procedure update_forensicmedicineandtoxicologyexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Forensic Medicine and Toxicology', achievementCategoryId, userIds);
END;
$$;

alter procedure update_forensicmedicineandtoxicologyexpert(bigint, bigint[]) owner to learner;
